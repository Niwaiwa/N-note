###### tags: `MongoDB` `Tuning`

# Install Mongodb Cluster (Version 4.2)

此說明為安裝MongoDB Cluster的步驟, 依照步驟安裝即可在短時間完成整個MongoDB Cluster的建置  

※此安裝script檔案只對 *Centos* 有用, 雖大部分步驟跟ubuntu相同, 但有些流程不一樣, 因此不可混用  

## Install MongoDB Cluster

依照指示步驟建立config、shard、router, 建立順序為以下, 順序錯誤不肯定會不會遇到什麼問題

0. Prepare username & password & Host & set performance
1. Install Config Server Secondary(2~)
2. Install Config Server Primary(1)
3. Install Shard Server Secondary(2~)
4. Install Shard Server Primary(1)
5. Repeat 3~4 untill installed all shard server
6. Install Router Server(Mongos), mongos各自獨立安裝
7. 檢查 /var/log/mongodb/mongos.diagnostic.data/ 資料夾是否已建立

※即使依照步驟安裝也有可能第一台不是Primary  
※如果曾經成功完成一次replica, 要重建replica就一定要清掉原本的所有成員重做, 不然就是要用不同_id才行  
※原本使用js檔案作為input可行, 但依照情況可能不行(碰運氣?), 所以全部使用EOF的方式  

    sh ./install_mongo_centos_ssl.sh

如果要建立有SSL的cluster就帶上參數  

    sh ./install_mongo_centos_ssl.sh --ssl

---

## Prepare Parameter

事先準備安裝需要的參數  

### Host, 追加到Cluster的所有機器上 /etc/hosts

```
10.1.1.121  mongo-router-1
10.1.1.122  mongo-router-2
10.1.1.123  mongo-config-1
10.1.1.124  mongo-config-2
10.1.1.125  mongo-config-3
10.1.1.126  mongo-shard-1-1
10.1.1.127  mongo-shard-1-2
10.1.1.128  mongo-shard-1-3
10.1.1.129  mongo-shard-2-1
10.1.1.132  mongo-shard-2-2
10.1.1.133  mongo-shard-2-3
```

### 各Member

```
config replSetName: config-svr
config repl_1: mongo-config-1
config repl_2: mongo-config-2
config repl_3: mongo-config-3
port: 27017

shard_1 replSetName: shard-1
shard_1 repl_1: mongo-shard-1-1
shard_1 repl_2: mongo-shard-1-2
shard_1 repl_3: mongo-shard-1-3
port: 27017

shard_2 replSetName: shard-2
shard_2 repl_1: mongo-shard-2-1
shard_2 repl_2: mongo-shard-2-2
shard_2 repl_3: mongo-shard-2-3
port: 27017

router_1: mongo-router-1
router_2: mongo-router-2
port: 27017
```

### Default username & password

如需修改預設帳密, 修改 install_mongo.sh 檔案
※注意整個Cluster最好有一個共通的帳密

```
username=""
password=""
```

### Uninstall All

執行後即會清空所有Mongos & Mongod的檔案跟設定

    sh ./uninstall_mongo.sh

### 機器參數調整

執行後即會自動設定基本的優化參數
※要查詢當前參數狀態請帶上 -l
※比較有可能有問題的是fstab跟io scheduler會因為機器硬碟配置而無法直接設定完成, 此時請參考下方參數調整章節

    system_configuration.sh

### Generate key file

可先產生mongodb使用的keyfile, 也可使用預先產生的

    openssl rand -base64 756 > keyfile

### 各種設定檔位置

路徑           | 說明
--------------|:-----
/etc/mongod.conf    | Mongod (config, shard)設定檔
/etc/mongos.conf    | Mongos (router)設定檔
/usr/lib/systemd/system/mongos.service  | Mongos system service 檔案位置
/var/log/mongodb/mongod.log  | Log檔案位置
/var/lib/mongo  | 實際資料位置
/var/log/mongodb/mongos.diagnostic.data/ | 監控用資料位置(diagnosticDataCollectionDirectoryPath), 權限700 mongod:mongod, 若無資料夾務必手動建立

### 各種預設設定

```
verbosity: 0  # systemLog 等級(預設)
port: 27017  # Bind Port
bindIp: 0.0.0.0  # Bind IP
```

---

## Sharded Cluster Administration 管理員知識

### Restart a Sharded Cluster

1. Disable the Balancer
    1. sh.stopBalancer()
2. Stop mongos routers
    1. systemctl stop mongos.service
3. Stop each shard replica set (先從secondary開始關)
    1. systemctl stop mongod.service
4. Stop config servers (先從secondary開始關)
    1. systemctl stop mongod.service

### Start Sharded Cluster

1. Start config servers & Check status
    1. systemctl start mongod.service
    2. rs.status()
2. Start each shard replica set & Check status
    1. systemctl start mongod.service
    2. rs.status()
3. Start mongos routers
    1. systemctl start mongos.service
4. Re-Enable the Balancer & Check status
    1. sh.startBalancer()
    2. sh.status()

### Update a Sharded Cluster version (4.2.10 -> 4.2.18)

0. 請先關閉其他使用db的服務並且執行完備份後再執行升級動作
1. Disable the Balancer (mongos)
    1. sh.stopBalancer()
2. reinstall config servers (先從secondary開始)
    * 如果為primary則需先執行以下步驟, 實行降級, 確認已降級為secondary
        1. rs.stepDown(10)
        2. rs.status()
    1. yum install -y mongodb-org
    2. mongo --version (確認版本)
    3. systemctl status mongod.service (確認運作是否正常)
    4. rs.status() (確認replica運作是否正常)
    5. db.version() (確認版本)
3. Stop each shard replica set (先從secondary開始關)
    * 如果為primary則需先執行以下步驟, 實行降級, 確認已降級為secondary
        1. rs.stepDown(10)
        2. rs.status()
    1. yum install -y mongodb-org
    2. mongo --version (確認版本)
    3. systemctl status mongod.service (確認運作是否正常)
    4. rs.status() (確認replica運作是否正常)
    5. db.version() (確認版本)
4. Stop mongos routers (任一台開始即可)
    1. yum install -y mongodb-org
    2. mongo --version (確認版本)
    3. systemctl status mongos.service (確認運作是否正常)
    4. systemctl status mongod.service (確認是否沒有運作)
        * systemctl disable mongod.service (確認是否沒有運作)
    5. systemctl restart mongos.service (理論上mongos需手動執行重啟)
    6. sh.status() (確認shard cluster運作是否正常)
    7. db.version() (確認版本)
5. 檢查服務連線db是否都有正常運作

### Backup & Restore MongoDB

Backup 備份不能直接過濾指定的collection, 使用gzip有可能導致restore失敗, 連接到cluster的時候使用uri會有很多參數上的衝突, 使用下面例子比較單純或自行寫scripts
※注意restore cluster不能restore config db, 需移除

```
mongodump -h 10.1.1.113:27017 -u '' -p '' --authenticationDatabase admin --excludeCollectionsWithPrefix=audit_log -o dump/
mongodump -h 10.1.1.113:27017 -u '' -p '' --authenticationDatabase admin -o dump/
mongodump -h 10.1.1.113:27017 -u '' -p '' --authenticationDatabase admin -o dump/ --numParallelCollections=8

mongodump --host test-shard-00-00-xzjyi.mongodb.net:27016 --ssl --username '' --password '' --authenticationDatabase admin --db pffaker --excludeCollectionsWithPrefix=audit_log --excludeCollectionsWithPrefix=event_async -o ./dump
```

Restore

```
mongorestore -h 10.1.1.121:27017 -u '' -p '' dump/ --nsExclude whale_old.audit_log --nsExclude event_old.event_async  --nsExclude whale.audit_log --nsExclude event.event_async --authenticationDatabase admin

mongorestore -h 10.1.1.121:27017 -u '' -p '' dump/ --authenticationDatabase admin --numParallelCollections=8
```

---

## 建立Cluster時的機器參數調整

參數調整須設定給所有shard, 設定完調整清單內的設定, 必須重開機確認沒有問題, 避免以後設定消失  
可執行system_configuration.sh來一鍵完成, 執行後須reboot  

※ 須注意如果執行過system_configuration.sh, 會導致/etc/fstab有重複的noatime, 因此要小心不要重複按, 若需要重新執行, 請以/etc/fstab_bak 備份檔案回復原始設定  

※ 須注意SSD drives設定會根據當前機器而有可能不是sda, 也同時有可能不會是sdc而是帶有數字的sdc1, 這時候就要手動處理  

### 調整清單

1. NTP (需在所有機器上都有, 為的是同步時間, 這項全交給MIS設定)
2. Filesystem: XFS (只能在創建機器時設定, wiredTiger engine用XFS效能較高)
3. nofile, nproc 64000 (mongod.service已經預設為此數量)
4. 禁用THP (MongoDB 建議禁用, 理由是效能問題)
5. 設定sysctl (kernel效能優化)
6. 禁用NUMA (參照下方說明)
7. SSD drives (參照下方說明)
8. DB 資料夾位置設定noatime (參照下方說明)

#### 確認當前filesystem

```
df -hT
```

#### 設定禁用THP

* 確認當前system版本(ubuntu or centos)

```
hostnamectl
```

* 設定禁用THP(transparent-hugepages), 以下為mongodb官方說明的ubuntu的設定方式

```
cat > /etc/init.d/disable-transparent-hugepages <<EOF
#!/bin/bash
### BEGIN INIT INFO
# Provides:          disable-transparent-hugepages
# Required-Start:    $local_fs
# Required-Stop:
# X-Start-Before:    mongod mongodb-mms-automation-agent
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: Disable Linux transparent huge pages
# Description:       Disable Linux transparent huge pages, to improve
#                    database performance.
### END INIT INFO

case \$1 in
  start)
    if [ -d /sys/kernel/mm/transparent_hugepage ]; then
      thp_path=/sys/kernel/mm/transparent_hugepage
    elif [ -d /sys/kernel/mm/redhat_transparent_hugepage ]; then
      thp_path=/sys/kernel/mm/redhat_transparent_hugepage
    else
      return 0
    fi

    echo 'never' | tee \${thp_path}/enabled > /dev/null

    unset thp_path
    ;;
esac
EOF

chmod 755 /etc/init.d/disable-transparent-hugepages
/etc/init.d/disable-transparent-hugepages start
# centos更新用 chkconfig --add disable-transparent-hugepages
update-rc.d disable-transparent-hugepages defaults

# 只在centos上直接關閉tuned, 不關閉會影響readahead設定被蓋掉
tuned-adm off
tuned-adm list
systemctl stop tuned
systemctl disable tuned

# 檢查當前設定
cat /sys/kernel/mm/transparent_hugepage/enabled
```

#### 設定sysctl

kernel參數調整, 有助提升效能

```
# 寫檔方式, 只用在/etc/sysctl.d 資料夾存在時
cat > /etc/sysctl.d/mongodb-sysctl.conf <<EOF
net.core.somaxconn=4096
net.ipv4.tcp_fin_timeout=30
net.ipv4.tcp_keepalive_intvl=30
net.ipv4.tcp_keepalive_time=120
net.ipv4.tcp_max_syn_backlog=4096
vm.swappiness=1
vm.dirty_ratio=15
vm.dirty_background_ratio=5
EOF

#指令方式
sysctl -w net.core.somaxconn=4096
sysctl -w net.ipv4.tcp_fin_timeout=30
sysctl -w net.ipv4.tcp_keepalive_intvl=30
sysctl -w net.ipv4.tcp_keepalive_time=120
sysctl -w net.ipv4.tcp_max_syn_backlog=4096
sysctl -w vm.swappiness=1
sysctl -w vm.dirty_ratio=15
sysctl -w vm.dirty_background_ratio=5

sysctl -p  # 上述設定生效

# vm.swappiness  # 預設60, 不設定為0是因為有bug
# vm.dirty_ratio  # 預設20
# vm.dirty_background_ratio  # 預設10
# net.*  # 基本上以官方跟常用調教方式來調整
# 使用sysctl -a | grep xxx 來檢查是否設定完成
```

#### 設定SSD drives參數

預設使用sda代號, 但根據機器有可能不會是sda, 且同時有可能會帶有數字sda1, 此時要手動處理

```
# 檢查Read-ahead是否為32, 這個參數跟熱數據 使用cache(緩存)的使用量有關, 太大會浪費記憶體
blockdev --getra /dev/sda

# 檢查IO Scheduler是否為"deadline"或"noop", 目前都使用VM, 因此用noop, sda為安裝系統時設定的硬碟代號, 依照安裝設定有可能不同
cat /sys/block/sda/queue/scheduler

# 只設定Read-ahead
blockdev --setra 32 /dev/xvdf

# 同時設定
echo 'ACTION=="add|change", KERNEL=="sda", ATTR{queue/scheduler}="noop", ATTR{bdi/read_ahead_kb}="16"' > /etc/udev/rules.d/60-sda.rules
```

#### DB 資料夾位置設定noatime

減少atime的更新, 增加效能

```
# 如果無其他mount狀態(例如dbpath是另外mount), 則直接根據根目錄的設定做更改
# 修改/etc/fstab, 在defaults後面追加noatime(也有追加更多參數的方式, 但先照官方說明只追加noatime)
UUID=7dce3c1f-85f6-4705-a338-b440c44ea5c7 /               xfs     defaults,noatime        0       0

# 使用sed直接替換, 正常不會有根目錄 跟 swap以外的資訊, 如果有就需要手動更新
sed -i 's/defaults/defaults,noatime/g' /etc/fstab
```

#### 禁用NUMA

非統一內存訪問, 通常為沒開啟, 可以檢查是否有開啟, 有開啟的話就將其關閉

```
dmesg | grep -i numa # 如果有node0以外的, 就表示numa没有关闭

apt-get install -y numactl
numastat  # 如果有node0以外的, 就表示numa没有关闭
numactl --hardware  # 输出结果 available: 1 nodes 如果大于1个nodes, 就表示numa没有关闭

# 如有開啟NUMA, 即需要調整為以下啟動方式
numactl --interleave=all mongod <options here>
```

## 資料來源

* <https://www.percona.com/blog/2016/08/12/tuning-linux-for-mongodb/>
* <https://docs.mongodb.com/v4.2/administration/production-checklist-operations/>
* <https://www.ibm.com/support/knowledgecenter/SS8G7U_19.4.0/com.ibm.app.mgmt.doc/content/install_storage_cassandra.html>
* <https://qiita.com/Nobuo/items/ee406deeea4fbfd83388>
