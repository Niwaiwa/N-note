###### tags: `Docker`

# Docker Swarm Cluster

## Docker swarm 參考架構

### DNS 服務發現(service discovery)

在任意的同一網路內的所有容器都會註冊service name(或network alias)到dns server且能夠互相找到對方  
* docker_DNS_service_discovery.jpg

### Docker swarm 內部負載平衡

```
# 查詢docker 註冊在linux的port跟轉發位置
iptables -L -t nat

# 轉發流程
1. request
2. service discovery
3. service vip(virtual ip)
4. ipvs load balancing
5. individual task(container)
```

### Docker swarm 外部負載平衡 (Swarm Mode Routing Mesh)

基本同內部負載平衡, 當request進入機器, 當這台機器沒有目標服務  
linux的 IPVS 負載平衡器 會將 ingress overlay network 上的request 重定向到有服務的機器上, 最終導到目標服務  
* docker_swarm_load_balancing_traffic_path.jpg
* docker_swarm_routing_mesh.jpg

### swarm cluster 所需 port

所有要串連swarm cluster的機器必須有以下port互通, 需確認是否有防火牆擋住

```
2377 (TCP)     為manager互相溝通的port, 當只有一個leader無其他reachable節點則不會用此port溝通
7946 (TCP/UDP) 為內部確認節點是否活著(heartbeat, ping), 而使用的port(無法更改)
4789 (UDP)     為swarm overlay網路互相溝通時需要的port, 從其他節點轉發request時會用到這個port
50   (TCP/UDP) 為swarm啟用加密時會使用的port, 未使用加密則不需要理會此port
```

---

### 跨網路(跨機房)的注意事項

通常只要節點間有互相開放必要的port就沒問題, 但要注意是否節點有在NAT後面  
假設跨機房, 需走外部IP, 此時有在NAT後面的節點, 需注意該節點被訪問服務時  
是否可以正常轉發request到其他節點, 根據NAT設定有可能NAT會取代來源IP, 導致無法根據來源IP返回response

#### 建立跨網路swarm (跨機房)

1. docker swarm init --advertise-addr 202.133.245.176:2377
2. docker swarm join --advertise-addr 202.133.245.178:2377 --token {token} 202.133.245.176:2377

- --advertise-addr, init跟join都指定才會走指定的public ip, 否則會自動抓ip(通常會抓到內部ip)
- --data-path-addr, init或join指定才會走指定的public ip, 否則會以--advertise-addr的ip為準, 因此可以不設定
- --data-path-port, init時指定會對整個cluster使用指定的port溝通, 否則走預設值
- docker swarm leave -f 強制離開swarm cluster

#### 測試用service

測試可以用簡單的nginx服務佈署進swarm

```
version: "3.6"

services:
  web:
    image: nginx
    ports:
      - "8080:80"
    command: [nginx-debug, '-g', 'daemon off;']
```

#### 查網路是否有通的工具

建立使用外部IP的swarm時, 需確認是否各port有互通  
swarm內部轉發是用ingress

##### tcpdump 監控即時流量

```
UDP 4789:
tcpdump -i ens160 udp port 4789 -nn

TCP/UDP 7946
tcpdump -i ens160 port 7946 -nn
```

##### nc 測試port是否有通

```
TCP:
nc 202.133.245.177 2377 -vz

UDP:
nc 202.133.245.177 7946 -vzu
```

---

### 災難復原

1. 任一節點斷網或不可復原
    - 原地復原, 回到cluster, 若無自動重新平衡則需手動平衡
    - 重新建立機器加入cluster

2. 節點斷網或不可復原, 數量超過容錯能力, 服務繼續運行, 但無法管理cluster
    - 故障節點原地復原, 回到cluster, 若無自動重新平衡則需手動平衡
    - 故障節點均無法復原, 需強制重新啟用cluster, 重新加入現有節點跟重建故障機器, 並重新加入cluster

3. 當最初init的節點非主節點(leader), 出現節點故障時, 即依照第2點情況處理, 當最初init的節點為leader且無其他leader, 當其他節點故障時, 還是可以正常管理cluster
    - 不可復原的情況, 通常是有複數的節點擁有管理權限(reachable, 最多七個), 且有決定權的節點從3個變為2個的時候會觸發無法操作cluster的狀態

### Docker Swarm 容錯能力

※當出現故障變為只有兩個節點, cluster服務繼續運行, 但無法更改cluster, 但要看是否是上述第3點的情況

Swarm Size | Majority| Fault Tolerance
-----------|:--------|:---------------
1     | 1     | 0
2     | 2     | 0
3     | 2     | 1
4     | 3     | 1
5     | 3     | 2
6     | 4     | 2
7     | 4     | 3
8     | 5     | 3
9     | 5     | 4

### 災難復原管理指令

#### 強制重新初始化節點(只用在故障節點無法恢復時)

```
docker swarm init --force-new-cluster
```

#### 強制重新平衡服務(會依序重啟所有服務)

```
docker service update --force ${service_name}
```

---

## Docker常用指令

### docker 常用指令

Command | Description
-----------|:--------
docker exec | Run a command in a running container
docker logs | Fetch the logs of a container
docker ps | List containers
docker start | Start one or more stopped containers
docker stop | Stop one or more running containers
docker rm | Remove one or more containers
docker images | List images

#### 維護管理用指令

Command | Description
-----------|:--------
docker container | Manage containers
docker info | Display system-wide information
docker inspect | Return low-level information on Docker objects
docker kill | Kill one or more running containers
docker restart | Restart one or more containers
docker rmi | Remove one or more images

#### build image常用指令

Command | Description
-----------|:--------
docker build | Build an image from a Dockerfile
docker login | Log in to a Docker registry
docker logout | Log out from a Docker registry
docker pull | Pull an image or a repository from a registry
docker push | Push an image or a repository to a registry
docker tag | Create a tag TARGET_IMAGE that refers to SOURCE_IMAGE

#### 常執行來清理未使用data的指令(prune)

Command | Description
-----------|:--------
docker image | Manage images
docker system | Manage Docker
docker volume | Manage volumes

#### 其他指令

Command | Description
-----------|:--------
docker cp | Copy files/folders between a container and the local filesystem
docker version | Show the Docker version information
docker top | Display the running processes of a container
docker stats | Display a live stream of container(s) resource usage statistics

---

### swarm 常用指令

Command | Description
-----------|:--------
docker node | Manage Swarm nodes
docker network | Manage networks
docker service | Manage services
docker stack | Manage Docker stacks
docker swarm | Manage Swarm

#### docker netwrok 指令

Command | Description
-----------|:--------
docker network connect | Connect a container to a network
docker network create | Create a network
docker network disconnect | Disconnect a container from a network
docker network inspect | Display detailed information on one or more networks
docker network ls | List networks
docker network prune | Remove all unused networks
docker network rm | Remove one or more networks

#### docker node 指令

Command | Description
-----------|:--------
docker node demote | Demote one or more nodes from manager in the swarm
docker node inspect | Display detailed information on one or more nodes
docker node ls | List nodes in the swarm
docker node promote | Promote one or more nodes to manager in the swarm
docker node ps | List tasks running on one or more nodes, defaults to current node
docker node rm | Remove one or more nodes from the swarm
docker node update | Update a node

#### docker stack 指令

Command | Description
-----------|:--------
docker stack deploy | Deploy a new stack or update an existing stack
docker stack ls | List stacks
docker stack ps | List the tasks in the stack
docker stack rm | Remove one or more stacks

#### docker service 指令

Command | Description
-----------|:--------
docker service create | Create a new service
docker service inspect | Display detailed information on one or more services
docker service logs | Fetch the logs of a service or task
docker service ls | List services
docker service ps | List the tasks of one or more services
docker service rm | Remove one or more services
docker service rollback | Revert changes to a service’s configuration
docker service scale | Scale one or multiple replicated services
docker service update | Update a service

## 資料來源

- https://docs.docker.com/engine/reference/run/
- https://docs.docker.com/compose/compose-file/
