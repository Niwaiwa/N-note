###### tags: `MongoDB` `Tuning`

# Mongodb Admin 管理&常用指令

管理MongoDB Cluster所需的一切概念, 知識, 作法等  

```js
# shard管理(mongos)
sh.status(true)
sh.stopBalancer()
sh.startBalancer()
db.printShardingStatus()
show users

# server狀態, database狀態等
db.stats()
db.stats(1024*1024)
db.serverStatus(1024*1024)
db.getFreeMonitoringStatus()
db.serverStatus()
db.adminCommand( { getDiagnosticData : 1 } )
db.getSiblingDB('admin').runCommand({"getDiagnosticData": 1})
db.getSiblingDB('admin').runCommand({"replSetGetStatus": 1})
db.getSiblingDB('admin').runCommand({"serverStatus": 1})
db.setProfilingLevel(1)
db.getProfilingLevel()
db.getProfilingStatus()

# replica set 管理
rs.status(true)
rs.stepDown(120)
rs.syncFrom()
db.printReplicationInfo()
db.printSlaveReplicationInfo()

# 建立role
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

# 建立user
use admin
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

db.createUser(
  {
    user: "mongodb_exporter",
    pwd: "nq40EsbqEnup",
    roles: [
        { role: "explainRole", db: "admin" },
        { role: "clusterMonitor", db: "admin" },
        { role: "read", db: "local" }
    ]
  }
)
show users
```

## Cluster管理

### 調整replica set優先權(priority)

調整選舉優先權, 任意情況需重新選舉時, 會根據優先權進行選舉, 優先權愈低, 愈不會成為primary, 可調整不同數據中心的優先權

```js
cfg = rs.conf()

cfg.members[0].priority = 0.5
cfg.members[1].priority = 2
cfg.members[2].priority = 2

rs.reconfig(cfg)

// priority = 0, 就不會成為primary
```

### 設定隱藏成員

隱藏成員不會被mongos判定為可以讀取的成員, 優先權必須為0, 但可以設為有投票權的成員

```js
cfg = rs.conf()
cfg.members[0].priority = 0
cfg.members[0].hidden = true
rs.reconfig(cfg)

// members[n].votes > 0, 才會被Write Concern計算為須確認寫入操作的成員
```

### 確認oplog size(bytes)

未設定時, 最大不超過硬碟 5% 或 50GB

```js
use local
db.oplog.rs.stats().maxSize
```

### 維護replica set members

維護secondary

```
# 使用mongo shell 安全地關閉mongod
db.shutdownServer()

# 設定檔註解跟加入設定, 成為獨立的mongod, 然後重啟mongod
net:
   bindIp: localhost,<hostname(s)|ip address(es)>
   port: 27218
#   port: 27018
#replication:
#   replSetName: shardA
#sharding:
#   clusterRole: shardsvr
setParameter:
   skipShardingConfigurationChecks: true
   disableLogicalSessionCacheRefresh: true

# 進行維護
mongo --port 27218

# 完成維護後, 安全地關閉mongod, 恢復原本的設定檔, 並重啟mongod
use admin
db.shutdownServer()

# 重啟mongod後, 確認member是否已從RECOVERING狀態變成SECONDARY狀態
rs.status()
```

維護primary

```
# 重新選舉primary, 並避免原本primary再次成為primary
rs.stepDown(300)

# 設定檔註解跟加入設定, 成為獨立的mongod, 然後重啟mongod
net:
   bindIp: localhost,<hostname(s)|ip address(es)>
   port: 27218
#   port: 27018
#replication:
#   replSetName: shardA
#sharding:
#   clusterRole: shardsvr
setParameter:
   skipShardingConfigurationChecks: true
   disableLogicalSessionCacheRefresh: true

# 進行維護
mongo --port 27218

# 完成維護後, 安全地關閉mongod, 恢復原本的設定檔, 並重啟mongod
use admin
db.shutdownServer()

# 重啟mongod後, 確認member是否已從RECOVERING狀態變成SECONDARY狀態
rs.status()
```

### primary強制退出primary

```js
db.adminCommand({replSetStepDown: 86400, force: 1})
rs.stepDown(120)
```

### 強制某個成員成為primary

```
mdb0.example.net - the current primary.
mdb1.example.net - a secondary.
mdb2.example.net - a secondary.

# 凍結mdb2.example.net
rs.freeze(120)

# mdb0.example.net退出primary
rs.stepDown(120)

# 兩分鐘內重新選舉後mdb1.example.net就會成為primary
```

## Cluster 增減成員

### 加入一個config(replica)新成員

先加入新成員, 但保持其為不可選舉狀態, 因為新的rs資料尚未完成同步

```js
rs.add( { host: "mongo-config-3:27017", priority: 0, votes: 0 } )
```

確認剛剛加入的新成員的狀態已為SECONDARY

```js
rs.status()
```

更新剛剛加入且已為SECONDARY狀態的成員為可選舉狀態

```
var cfg = rs.conf();
cfg.members[n].priority = 1;
cfg.members[n].votes = 1;
rs.reconfig(cfg)
```

移除非需要或有問題的成員

```js
rs.remove("mongo-config-3:27017")
```

### 加入一個新的shard

```
sh.addShard( "rs1/mongodb0.example.net:27017" )
```

### 建立新的Mongos

設定檔加入config位置

```
sharding:
  configDB: <configReplSetName>/cfg1.example.net:27019,cfg2.example.net:27019
```

啟動mongos後, 執行addShard指令

```js
sh.addShard( "<replSetName>/s1-mongo1.example.net:27018,s1-mongo2.example.net:27018,s1-mongo3.example.net:27018")
```

## 故障處理, 可能的機器fail狀況

是方機房到Linode Tokyo網路延遲大約50ms

### 單機器fail (可復原&不可復原)

1. shard-1的rs-1(primary)故障(可恢復的情況)

   1. Cluster自動錯誤轉移, 重新選舉出新的primary
   2. 一定時間內rs-1自動或手動恢復後會先成為secondary, 待資料同步後, cluster會自行重新選舉, 因此rs-1不一定會重新成為primary
      - 現階段規格的oplog約為16GB(最大不超過5%或50GB), 再加上現在的資料量及資料增加趨勢, 應可以在超過6小時的時間後都還可以就地復原

2. shard-1的primary故障(不可恢復的情況)

   1. Cluster自動錯誤轉移, 重新選舉出新的primary
   2. 重建一台機器 & 重建一個MongoDB實例
   3. 重新加入cluster, 此時需要進行"初始同步", 完成後會成為secondary, 不一定會重新成為primary

3. shard-1的rs-2(secondary)故障(可恢復的情況)

   1. 自動或手動恢復後, 回到secondary狀態
      - 現階段規格的oplog約為16GB(最大不超過5%或50GB), 再加上現在的資料量及資料增加趨勢, 應可以在超過6小時的時間後都還可以就地復原

4. shard-1的rs-2(secondary)故障(不可恢復的情況)

   1. 重建一台機器 & 重建一個MongoDB實例
   2. 重新加入cluster, 等待同步完成或進行"初始同步", 完成後會成為secondary

5. Mongos 故障

   - 依照原本建立方式重建mongos並連接上cluster

### 單shard fail

1. 任意shard故障(不管是否可恢復)

   - 整個cluster須停止並從備份重建

### 網路分區(兩個機房)

1. 網路分區(network partition), 兩個機房的配置, 斷線超過10秒
   - 五個replica, 網路分區後, 少數成員的網路區域不會自動重選出primary, 因判斷為少數成員
   - 如觸發了錯誤轉移(fail over)重新選舉新的primary, 短時間內從網路分區恢復後, 會自動rollback舊的primary

2. 網路分區(network partition), 兩個機房的配置, 斷線不超過10秒
   - 不會觸發錯誤轉移(fail over), 自動互相重新確認後正常運行

3. 網路分區(network partition), 兩個機房有相同數量成員的配置, 斷線超過10秒

   - 會導致雙方互相判斷對方都是多數, 而雙方都進入read only狀態

4. 網路分區(network partition), 兩個機房有相同數量成員但linode有多一個arbiter, 斷線超過10秒

   - 會導致linode為多數並進行錯誤轉移選出primary, 但是方機房判斷為少數, 而進入read only狀態

5. 網路分區(network partition), 兩個機房有相同數量成員但linode有多一個arbiter且其中一個成員無投票權, 斷線超過10秒

   - 會導致雙方互相判斷對方都是多數, 而雙方都進入read only狀態

6. 網路分區(network partition), 三個機房221配置, 其中一個機房斷線超過10秒

   - 任一機房故障, 剩下的都一定為多數, 自動進行故障轉移

7. 網路分區(network partition), 三個機房221配置, 第三機房的1為仲裁器, 其中一個機房斷線超過10秒

   - 任一機房故障, 剩下的都一定為多數, 自動進行故障轉移
   - ※官方不建議在評估使用write_concern時使用仲裁器, 故偏向使用一般的replica

8. 網路分區(network partition), 是方機房有11個成員, linode有7個成員(兩個機房架構), 斷線超過10秒

   - 會導致linode判斷對方都是多數, 而進入read only狀態

9. linode備援機器故障

   - 重建備援機器, 執行初始同步

### Logrotate

#### 原生

尋找官方設定(有, 但目前未使用)

#### Linux logrotate module

vi /etc/logrotate.d/mongo

```conf
/var/log/mongodb/*.log {
    missingok
    notifempty
    dateext
    delaycompress
    copytruncate
    rotate 30
    compress
    daily
    create 0600 root root
}
```

#### 測試是否設定正確, 正常的話無回應

logrotate -f /etc/logrotate.d/mongo
logrotate -f /etc/logrotate.conf

#### Linux logrotate copytruncate

原理為不重新命名檔案, 先copy資料到新檔案再truncate原本的資料
根據頻率資料很少的情況下, 不太會遇到log遺失的情況
如不使用copytruncate 則可參考下面的設定方式

* https://www.percona.com/blog/2018/09/27/automating-mongodb-log-rotation/
