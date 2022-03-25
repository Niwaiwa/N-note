###### tags: `Portfolio(作品集)`

# Gitlab CI/CD & Docker Swarm

建立gitlab版控軟體, 建立使用gitlab CI/CD & Docker Swarm的CI/CD範例

## 安裝步驟

1. 準備三台VM (網路都正常互通)
2. 在第一台機器上安裝docker & docker compose
3. 將gitlab compose file放到第一台機器上(Gitlab docker compose file)
4. 啟動gitlab服務(等待一段時間後登入使用)
5. 再第二台跟第三台機器上安裝docker & docker compoes & 啟動Docker Swarm & 互相註冊連通
6. 在swarm任一台上啟動私有容器倉庫(registry)
7. swarm任一台上設置/etc/docker/daemon.json檔案並填入insecure-registries設定後重啟docker服務
8. 依照gitlab專案的CI/CD裡面安裝runner的步驟安裝gitlab runner到swarm機器上(第二第三台)
9. 創建gitlab專案
10. 之後在swarm機器上註冊runner到gitlab(網路要正常互通才會正常運作, 且要注意使用的ip或domain)
11. 在專案上準備服務要用的code跟檔案
12. 在專案上撰寫CI檔案(.gitlab-ci.yml)
13. push CI檔案之後查看gitlab專案的CI/CD是否有正常運作
14. 排除問題


### Gitlab

初始啟動會需要等待一段時間, 初始管理者密碼預先設定到compose file, 預設port調整

gitlab compose file
```
version: '3'
services:
 
  gitlab:
    image: 'gitlab/gitlab-ce:latest'
    restart: always
    hostname: 'gitlab.example.com'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://10.1.30.31:8929'
        gitlab_rails['gitlab_shell_ssh_port'] = 9955
        gitlab_rails['initial_root_password'] = 'password'
    ports:
      - '8929:8929'
      - '9955:22'
    # volumes:
    #   - './gitlab/config:/etc/gitlab'
    #   - './gitlab/logs:/var/log/gitlab'
    #   - './gitlab/data:/var/opt/gitlab'
```


使用shell模式並且需要呼叫docker指令的情況下須調整權限(如果需重啟則重啟VM)
Runner permission
```
sudo groupadd docker
usermod -aG docker gitlab-runner
#reboot
```

若遇到CI執行shell錯誤, 則需要調整swarm機器的 /home/gitlab-runner/.bash_logout 檔案, 將其刪除或者將內容全部註解掉, 理論上即可解決問題
```
ERROR: Job failed (system failure): prepare environment: exit status 1. Check https://docs.gitlab.com/runner/shells/index.html#shell-profile-loading for more information
```

若遇到CI執行git checkout錯誤, 則需要重設定gitlab服務的external_url, 改為指定ip
```
fatal: unable to access 'http://gitlab.example.com:8929/test/test1.git/': Could not resolve host: gitlab.example.com
```

### Gitlab 專案

nginx.conf
```
user  nginx;
worker_processes auto;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    use epoll;
    multi_accept on;
    worker_connections 65535;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    proxy_set_header X-Forwarded-For $http_x_forwarded_for;
    proxy_set_header X-Real-IP $http_x_real_ip;

    log_format  main  '[$http_x_real_ip] $remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;
    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout 120;
    client_max_body_size 100m;
    fastcgi_connect_timeout 180;
    fastcgi_read_timeout 180;
    #gzip  on;
    include /etc/nginx/conf.d/*.conf;
}
```

ark.conf
```
server {
    listen 80;

    location / {
        add_header Content-Type text/html;

        return 200 '<html><body>Test 4</body></html>';
    }
}
```

DockerfileNginx
```
FROM nginx:1.19-alpine

COPY docker/nginx/nginx.conf /etc/nginx/nginx.conf
COPY docker/nginx/ark.conf /etc/nginx/conf.d/default.conf

```

docker-swarm-release.yml
```
version: '3.6'

networks:
  ark-pro-network:

services:
  nginx:
    image: 10.1.30.37:5000/release_ark_pro_nginx:latest
    ports:
      - 9500:80
    networks:
      ark-pro-network:
    deploy:
      mode: replicated
      replicas: 2
      update_config:
        parallelism: 1
        order: start-first
    healthcheck:
      test: ["CMD-SHELL", "netstat -tulpn | grep nginx || exit 1"]
      interval: 30s
      timeout: 30s
      retries: 3
```

.gitlab-ci.yml
```
stages:
  - prune
  - build
  - test
  - deploy


prune1:
  stage: prune
  tags:
    - test1
  script:
    - docker system prune --force
    - docker volume prune --force
    - docker image prune --force

build1:
  stage: build
  tags:
    - test1
  script:
    - echo "Do your build here"
    - docker_registry_host="10.1.30.37:5000"
    - docker_tag_name="release_ark_pro"
    - docker build -t ${docker_tag_name}_nginx -f DockerfileNginx .
    - docker tag ${docker_tag_name}_nginx:latest ${docker_registry_host}/${docker_tag_name}_nginx:latest
    - docker push ${docker_registry_host}/${docker_tag_name}_nginx:latest

test1:
  stage: test
  tags:
    - test1
  script:
    - echo "Do a test here"
    - echo "For example run a test suite"

test2:
  stage: test
  tags:
    - test1
  script:
    - echo "Do another parallel test here"
    - echo "For example run a lint test"

deploy1:
  stage: deploy
  tags:
    - test1
  script:
    - echo "Do your deploy here"
    - docker_registry_host="10.1.30.37:5000"
    - stack_name="release_ark_pro"
    - docker pull ${docker_registry_host}/${stack_name}_nginx:latest
    - docker stack deploy -c docker-swarm-release.yml ${stack_name} &&
      docker service update -q --with-registry-auth --stop-grace-period 1m --image ${docker_registry_host}/${stack_name}_nginx:latest ${stack_name}_nginx

```

### Docker 安裝

可以參考官方的文件, 因有可能會因改版而有變動

Docker Engine
```
sudo apt update
sudo apt-get install     ca-certificates     curl     gnupg     lsb-release
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo   "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

Docker compose

```
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```

Docker registry

```
docker run -d -p 5000:5000 --name registry registry:2
```

Docker daemon.json

設定私有倉庫的非https連線, 設定後須重啟docker服務 (systemctl restart docker)
```
{
        "insecure-registries":["172.28.118.163:5000"]
}
```