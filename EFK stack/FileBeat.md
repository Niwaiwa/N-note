###### tags: `EFK`

# FileBeat

## 安裝(yum)

### 建立yum repo file, elastic.repo  

sudo rpm --import https://packages.elastic.co/GPG-KEY-elasticsearch  

```sh
echo "[elastic-7.x]
name=Elastic repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md" > /etc/yum.repos.d/elastic.repo
```

yum install filebeat-7.9.0 -y
systemctl enable filebeat
systemctl status filebeat

## 升級版本(大版號相同7.x)

大版號相同直接升級並重啟服務即可

yum install filebeat-7.11.1 -y
systemctl restart filebeat

### 打開mongodb module

filebeat modules enable mongodb

### 修改設定檔 /etc/filebeat/filebeat.yml

改index name要注意會連動index template跟ilm, 所以一定要正確設定, 否則ilm會失敗

```yml
# ================================== Template ==================================
setup.template.name: "filebeat-%{[agent.version]}"  # 不須調整名稱的話直接不填, 內容為預設值
setup.template.pattern: "filebeat-%{[agent.version]}-*"  # 不須調整名稱的話直接不填, 內容為預設值
# ====================== Index Lifecycle Management (ILM) ======================
setup.ilm.rollover_alias: "filebeat-%{[agent.version]}"  # 不須調整名稱的話直接不填, 內容為預設值
setup.ilm.policy_name: "filebeat"  # 不須調整名稱的話直接不填, 內容為預設值

output.elasticsearch:
  hosts: ["myEShost:9200"]
  index: "filebeat-%{[agent.version]}-%{+yyyy.MM.dd}"  # 不須調整名稱的話直接不填, 內容為預設值

# 若kibana跟elasticsearch設定的機器位置不一樣就需要指定, 或者不使用預設的就都不要設定, 如果有nginx在kibana前面, 可能就需要帳密
setup.kibana:
    host: "admin:password@myEShost:5601"
```

### 修改設定檔 /etc/filebeat/modules.d/mongodb.yml

指定LOG檔案的path

```yml
- module: mongodb
  # All logs
  log:
    enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    var.paths: ["/var/log/mongodb/mongos.log*"]
```

### 檢查設定檔是否符合格式

filebeat setup -e

### 啟動filebeat service

service filebeat start

## 刪除

systemctl stop filebeat.service
yum remove filebeat -y
rm -r /etc/filebeat
rm /etc/yum.repos.d/elastic.repo
