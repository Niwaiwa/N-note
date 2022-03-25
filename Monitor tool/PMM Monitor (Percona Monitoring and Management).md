###### tags: `PMM` `監控`

# PMM Monitor (Percona Monitoring and Management)

## PMM Server

建立PMM 開源監控Server, 先建立server做為資料蒐集中心, 再去各個要監控的機器上建立Client, 然後監控機器上的目標  
目前可監控的目標: MongoDB, Mysql, MariaDB, PostgreSQL, Node(Host)  

### Install pmm-server with docker

```sh
docker pull percona/pmm-server:2
docker create --volume /srv --name pmm-data percona/pmm-server:2 /bin/true
docker run --detach --restart always --publish 80:80 --publish 443:443 --volumes-from pmm-data --name pmm-server percona/pmm-server:2
```

### Backup pmm-data from pmm-server

```sh
mkdir pmm-data && cd pmm-data
docker cp pmm-data:/srv .
```

#### pmm-server 帳號 密碼 HOST PORT

```
username: admin
password: password
host: 10.1.1.130
port: 443
```

### Uninstall pmm-server

```sh
docker stop pmm-server
docker rm pmm-server
docker rm pmm-data
```

## PMM Client

1. 安裝PMM Client
2. 註冊PMM Server給PMM Client
3. 註冊要監控的service

### Install pmm-client with rpm(centos yum)

```sh
yum install https://repo.percona.com/yum/percona-release-latest.noarch.rpm -y
yum install pmm2-client -y
```

#### register pmm-server command for pmm-client

```sh
pmm-admin config --server-insecure-tls --server-url=https://admin:password@10.1.1.130:443
```

#### Uninstall pmm-client

```sh
yum remove pmm2-client -y
```

### 註冊要監控的service

```sh
#Shard-1
pmm-admin add mongodb --username=mongodb_exporter --password=password --service-name=mongo-4.2-rs1-01 --host=10.1.1.126 --port=27017 --cluster=mongo-4.2 --replication-set=shard-1 --environment=4.2
pmm-admin add mongodb --username=mongodb_exporter --password=password --service-name=mongo-4.2-rs1-02 --host=10.1.1.127 --port=27017 --cluster=mongo-4.2 --replication-set=shard-1 --environment=4.2
pmm-admin add mongodb --username=mongodb_exporter --password=password --service-name=mongo-4.2-rs1-03 --host=10.1.1.128 --port=27017 --cluster=mongo-4.2 --replication-set=shard-1 --environment=4.2

#Shard-2
pmm-admin add mongodb --username=mongodb_exporter --password=password --service-name=mongo-4.2-rs2-01 --host=10.1.1.129 --port=27017 --cluster=mongo-4.2 --replication-set=shard-2 --environment=4.2
pmm-admin add mongodb --username=mongodb_exporter --password=password --service-name=mongo-4.2-rs2-02 --host=10.1.1.132 --port=27017 --cluster=mongo-4.2 --replication-set=shard-2 --environment=4.2
pmm-admin add mongodb --username=mongodb_exporter --password=password --service-name=mongo-4.2-rs2-03 --host=10.1.1.133 --port=27017 --cluster=mongo-4.2 --replication-set=shard-2 --environment=4.2

#Config
pmm-admin add mongodb --username=mongodb_exporter --password=password --service-name=mongo-4.2-config-01 --host=10.1.1.123 --port=27017 --cluster=mongo-4.2 --replication-set=config-svr --environment=4.2
pmm-admin add mongodb --username=mongodb_exporter --password=password --service-name=mongo-4.2-config-02 --host=10.1.1.124 --port=27017 --cluster=mongo-4.2 --replication-set=config-svr --environment=4.2
pmm-admin add mongodb --username=mongodb_exporter --password=password --service-name=mongo-4.2-config-03 --host=10.1.1.125 --port=27017 --cluster=mongo-4.2 --replication-set=config-svr --environment=4.2

#Router
pmm-admin add mongodb --username=mongodb_exporter --password=password --service-name=mongo-4.2-mongos-01 --host=10.1.1.121 --port=27017 --cluster=mongo-4.2 --environment=4.2
pmm-admin add mongodb --username=mongodb_exporter --password=password --service-name=mongo-4.2-mongos-02 --host=10.1.1.122 --port=27017 --cluster=mongo-4.2 --environment=4.2 --custom-labels="host_type=mongos"
```

#### 如果DB有使用TLS

註冊要監控的service時加上參數, 使用TLS並且忽略domain & 憑證檢查

```
--tls --tls-skip-verify
```

#### 出現 duplicate match

編輯(edit), Metrics追加 type="mongos", 只會從server取出type=mongos的資料, 就不會重複

#### Percona Mongodb_exporter 啟動參數

參數           | 說明
---------------|:-----
--compatible-mode  | 統一拿出新舊參數, 會查詢所有DB, 在mongos上根據實際地理位置有可能會很慢, 因此有複數mongos時可以移除此參數不重複拿資料
--log.level="debug"  | 設定log等級, 除錯用 [debug, info, warn, error, fatal]
--mongodb.global-conn-pool  | 使用global連線池
--disable.replicasetstatus  | 不查詢replicasetstatus (replSetGetStatus), mongos不支援
--disable.diagnosticdata  | 此參數通常還是都會需要, 但要注意mongos有可能因為沒建立資料夾導致沒啟用而拿不到資料(diagnosticDataCollectionDirectoryPath)
--mongodb.uri=mongodb://user:pass@127.0.0.1:27017/admin?ssl=true  | v0.20.1 版本只支援 tls 的參數, 設定其他的都會被忽視掉

#### Percona Mongodb_exporter 查看 metrics

找到mongodb_exporter的port跟agent id

```
http://{IP}:{port}/metrics
pmm-admin list
帳號: pmm
密碼: {Agent ID}
```

#### 監控可能會用到的實際 mongo shell 參數

```js
db.getSiblingDB('admin').runCommand({"getDiagnosticData": 1})
db.getSiblingDB('admin').runCommand({"replSetGetStatus": 1})
db.getSiblingDB('admin').runCommand({"serverStatus": 1})
```

#### mongodb_exporter 監控用帳號 & 權限建立指令

```js
db.createUser(
  {
    user: "mongodb_exporter",
    pwd: "password",
    roles: [
        { role: "clusterMonitor", db: "admin" },
        { role: "read", db: "local" }
    ]
  }
)
```

或  

```js
db.createRole({
    role: "explainRole",
    privileges: [{
        resource: {
            db: "",
            collection: ""
            },
        actions: [
            "listIndexes",
            "listCollections",
            "dbStats",
            "dbHash",
            "collStats",
            "find"
            ]
        }],
    roles:[]
})

db.getSiblingDB("admin").createUser({
   user: "mongodb_exporter",
   pwd: "password",
   roles: [
      { role: "explainRole", db: "admin" },
      { role: "clusterMonitor", db: "admin" },
      { role: "read", db: "local" }
   ]
})
```

### 移除註冊過的service

```sh
pmm-admin remove mongodb mongo-4.2-rs1-01
pmm-admin remove mongodb mongo-4.2-rs1-02
pmm-admin remove mongodb mongo-4.2-rs1-03
pmm-admin remove mongodb mongo-4.2-rs2-01
pmm-admin remove mongodb mongo-4.2-rs2-02
pmm-admin remove mongodb mongo-4.2-rs2-03
pmm-admin remove mongodb mongo-4.2-config-01
pmm-admin remove mongodb mongo-4.2-config-02
pmm-admin remove mongodb mongo-4.2-config-03
pmm-admin remove mongodb mongo-4.2-mongos-01
pmm-admin remove mongodb mongo-4.2-mongos-02
```

### 加入外部的exporter

```sh
pmm-admin add external --listen-port=9216 --environment=4.2 --cluster=mongo-4.2 --service-name=mongo-router-3 --group=MongoDB
```

### 移除外部的exporter

```sh
pmm-admin remove external mongo-router-3
```

### Mariadb 監控

```
pmm-admin add mysql --username=pmm --password=password --query-source='perfschema' test-mysql 127.0.0.1:3306
```

MariaDB預設未開啟監控, 需在設定檔加入, 並重啟DB

```
innodb_monitor_enable=all
performance_schema=ON
userstat=1  # 未必需要(只有MariaDB有, Mysql沒有此參數)
```

## 資料來源

* <https://www.percona.com/doc/percona-monitoring-and-management/2.x/setting-up/server/docker.html>
* <https://www.percona.com/doc/percona-monitoring-and-management/2.x/setting-up/client/mongodb.html>
* <https://www.percona.com/blog/2020/09/22/new-mongodb-exporter-released-with-percona-monitoring-and-management-2-10-0/>
* <https://docs.mongodb.com/v4.2/administration/analyzing-mongodb-performance/>
