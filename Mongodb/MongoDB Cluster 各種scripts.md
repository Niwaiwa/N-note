###### tags: `MongoDB` `Tuning`

# MongoDB Cluster 各種scripts

## Linux system tuning

```bash
#!/usr/bin/env bash

DISK=""

CheckDisk(){
    # df /var/lib/mongo
    TARGET_DISK=`lsblk -r | grep /var/lib/mongo | awk {'print $1'}`
    # if [ "$?" == 0 ]; then
    if [ ! -z "${TARGET_DISK}" ]; then
        DISK="${TARGET_DISK}"
    else
        DISK=`lsblk -r | grep / | awk {'print $1'}`
    fi
}

CheckOS(){
    source /etc/os-release
    case ${ID} in
    debian|ubuntu|devuan)
        ;;
    centos|fedora|rhel)
        ;;
    *)
        printf "${RED} 作業系統版本判斷失敗，OS 更新作業停止！ ${PLAIN}\n"
        kill -SIGINT $$
        ;;
    esac
}
CheckOS
CheckDisk
echo "Current OS: ${ID}"
echo "Current target disk: ${DISK}"
echo

if [ "$1" != "-l" ]; then
echo "Start set system configuration."
echo "disable transparent hugepages..."
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

if [ "${ID}" == "centos" ]; then
    chkconfig --add disable-transparent-hugepages
else
    update-rc.d disable-transparent-hugepages defaults
fi

# 只對centos有tuned的執行
if [ "${ID}" == "centos" ]; then
    # 直接禁用tuned
    tuned-adm off
    tuned-adm list
    systemctl stop tuned
    systemctl disable tuned
    # systemctl enable tuned
    # systemctl start tuned

#     echo "set centos tuned..."
#     mkdir -p /etc/tuned/virtual-guest-no-thp
#     cat > /etc/tuned/virtual-guest-no-thp/tuned.conf <<EOF
# [main]
# include=virtual-guest

# [vm]
# transparent_hugepages=never
# EOF
#     tuned-adm profile virtual-guest-no-thp
fi

echo "disable transparent hugepages... down"
sleep 1

echo "write sysctl variable..."
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
echo "write sysctl variable... down"
sleep 1

echo "set SSD Read-ahead and IO scheduler..."
if [ "${ID}" == "centos" ]; then
    # 使用lsblk查詢目標的KNAME(kernel name), centos設定無效的情況請禁用tuned
    # lsblk --output NAME,KNAME,TYPE,MAJ:MIN,FSTYPE,SIZE,RA,MOUNTPOINT,LABEL
    echo 'ACTION=="add|change", KERNEL=="'${DISK}'", ATTR{queue/scheduler}="noop", ATTR{bdi/read_ahead_kb}="16"' > /etc/udev/rules.d/60-${DISK}.rules
else
    echo 'ACTION=="add|change", KERNEL=="'${DISK}'", ATTR{queue/scheduler}="noop", ATTR{bdi/read_ahead_kb}="16"' > /etc/udev/rules.d/60-${DISK}.rules
fi
echo "set SSD Read-ahead and IO scheduler... down"
sleep 1

echo "set disk noatime..."
if [ -f "/etc/fstab_bak" ]; then
    cp /etc/fstab_bak /etc/fstab
else
    cp /etc/fstab /etc/fstab_bak
fi
# sed -i 's/defaults/defaults,noatime/g' /etc/fstab
# 只取代第一個match的, 通常為根目錄
sed -i '0,/defaults/s//defaults,noatime/' /etc/fstab
echo "set disk noatime... down"
sleep 1

echo "End set system configuration."
echo
echo "=============================="
# end ! -l
fi


echo "get transparent_hugepage..."
cat /sys/kernel/mm/transparent_hugepage/enabled
echo
if [ "${ID}" == "centos" ]; then
    echo "get tuned list, virtual-guest-no-thp"
    tuned-adm list | grep virtual-guest-no-thp
fi
echo

echo "get sysctl variable..."
sysctl -a | grep net.core.somaxconn
sysctl -a | grep net.ipv4.tcp_fin_timeout
sysctl -a | grep net.ipv4.tcp_keepalive_intvl
sysctl -a | grep net.ipv4.tcp_keepalive_time
sysctl -a | grep net.ipv4.tcp_max_syn_backlog
sysctl -a | grep vm.swappiness
sysctl -a | grep vm.dirty_ratio
sysctl -a | grep vm.dirty_background_ratio
echo

echo "get Read-ahead..."
blockdev --getra /dev/${DISK}
echo

echo "get IO scheduler..."
cat /sys/block/${DISK}/queue/scheduler
echo

echo "get noatime"
cat /etc/fstab
echo

echo "get numa"
dmesg | grep -i numa
echo

```

## Install MongoDB Cluster

```bash
#/bin/bash

Notify(){
    echo "$(date): $1 $2"
}

Install_MongoDB(){
    Notify ongoing "Install MongoDB Package..."
    echo "[mongodb-org-4.2]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/7Server/mongodb-org/4.2/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-4.2.asc" > /etc/yum.repos.d/mongodb-org-4.2.repo

    yum install -y mongodb-org
    # yum install -y mongodb-org-4.2.9 mongodb-org-server-4.2.9 mongodb-org-shell-4.2.9 mongodb-org-mongos-4.2.9 mongodb-org-tools-4.2.9

    # Notify ongoing "固定版本"
    # EXCLUDE_MONGO="exclude=mongodb-org,mongodb-org-server,mongodb-org-shell,mongodb-org-mongos,mongodb-org-tools"
    # if [ -z $(cat /etc/yum.conf | grep ${EXCLUDE_MONGO}) ]; then
    #     echo EXCLUDE_MONGO >> /etc/yum.conf
    # fi
}

SSL="$1"
SSL_MongoDB_Config(){
    if [ "$SSL" == "--ssl" ]; then
        SSL_PARAMETER="--tls --tlsAllowInvalidHostnames --tlsAllowInvalidCertificates "

        sed -i "33i \ \ tls:" $1
        sed -i "34i \ \ \ \ mode: requireTLS" $1
        sed -i "35i \ \ \ \ certificateKeyFile: /etc/ssl/mongodb.pem" $1
        sed -i "36i \ \ \ \ CAFile: /etc/ssl/mongodbCA_bundle.crt" $1
        sed -i "37i \ \ \ \ allowConnectionsWithoutCertificates: true" $1
        sed -i "38i \ \ \ \ allowInvalidCertificates: true" $1
        sed -i "39i \ \ \ \ allowInvalidHostnames: true" $1
    fi
}
if [ "${SSL}" != '--ssl' ]; then
    SSL="None"
fi

SSL_PARAMETER=""
PWD=`pwd`
echo "Current Path: ${PWD}"
echo "Current SSL: ${SSL}"

Notify start 開始MongoDB安裝流程

# echo -e "MongoDB Cluster 配置開始"
read -p "請輸入MongoDB 帳號: " username
printf "\n"
read -sp "請輸入MongoDB 密碼: " password
printf "\n"
read -sp "請再次輸入MongoDB 密碼: " password1
printf "\n\n"
# username=""
# password=""
# password1=""

if [[ $password != $password1 ]]; then
	echo '密碼不符合'
	exit 1
fi

read -p "請輸入欲建立的環境(test: 1, prod: 2): " envtype
printf "\n"

read -p "請輸入欲建立的MongoDB 類型(router: 1, config: 2, shard: 3): " mtype
printf "\n"

# mongos
if [[ $mtype == "1" ]]; then
    read -p "請輸入MongoDB config 成員有幾個(2 or 3...): " confignum
    printf "\n"
    read -p "請輸入MongoDB config 名稱: " configname
    printf "\n"
    # configname="config-svr"
    CONFIG_ARRAY=()
    cnfstr="${configname}\/"
    for ((i=1;i<=${confignum};i++))
    do
        read -p "請輸入MongoDB config server第 ${i} 個的位置 (IP or Host): " cnf_ip_1
        # printf "\n"
        read -p "請輸入MongoDB config server第 ${i} 個的Port (default 27017): " cnf_port_1
        if [ -z "${cnf_port_1}" ]; then
            cnf_port_1="27017"
        fi
        printf "${cnf_ip_1}:${cnf_port_1}\n"
        CONFIG_ARRAY+=("${cnf_ip_1}:${cnf_port_1}")
    done
    for ((i=0; i<${#CONFIG_ARRAY[@]}; i++))
    do
        cnfstr="${cnfstr}${CONFIG_ARRAY[i]},"
    done
    cnfstr="${cnfstr::-1}"
    echo "config: ${cnfstr}"

    ########################
    read -p "請輸入欲加入幾個MongoDB Shard 成員(1, 2, 3...): " num
    printf "\n"
    SHARD_ARRAY=()

    for ((x=1;x<=${num};x++))
    do
        read -p "請輸入MongoDB 第 ${x} 個 shard 有幾個成員 (2 or 3...): " shardnum
        printf "\n"
        # read -p "請輸入MongoDB 第 ${x} 個 shard 的名稱: " shardname
        # printf "\n"
        shardname="shard-${x}/"

        RS_ARRAY=()
        for ((i=1;i<=${shardnum};i++))
        do
            read -p "請輸入MongoDB shard server第 ${i} 個的位置 (IP or Host): " rs1
            # printf "\n"
            read -p "請輸入MongoDB shard server第 ${i} 個的Port (default 27017): " rsp1
            if [ -z "${rsp1}" ]; then
                rsp1="27017"
            fi
            printf "${rs1}:${rsp1}\n"
            RS_ARRAY+=("${rs1}:${rsp1}")
        done
        for ((i=0; i<${#RS_ARRAY[@]}; i++))
        do
            shardname="${shardname}${RS_ARRAY[i]},"
        done
        shardname="${shardname::-1}"
        SHARD_ARRAY+=(${shardname})

    done
    # SHARD_ARRAY=("shard-1/10.1.1.126:27017,10.1.1.127:27017" "shard-2/10.1.1.128:27017,10.1.1.129:27017")
    # #=array長度
    # for((i=0; i<${#SHARD_ARRAY[@]}; i++)) do done
    # for shardinfo in "${#SHARD_ARRAY[@]}"
    # for i in {1..${num}}
    for ((i=0; i<${#SHARD_ARRAY[@]}; i++))
    do
        num=$((i+1))
        echo "shard-${num}: ${SHARD_ARRAY[i]}"
    done
fi

# config
if [[ $mtype == "2" ]]; then
    read -p "請輸入欲建立幾個MongoDB ReplicaSet 成員(1 or 2...): " confignum
    printf "\n"
    read -p "請輸入欲建立第幾個的MongoDB config (1 or 2...): " cnfnum
    printf "\n"
    if [[ $cnfnum == "1" ]]; then
        # read -p "請輸入MongoDB config 成員有幾個(2 or 3...): " confignum
        # printf "\n"
        read -p "請輸入MongoDB config 名稱: " configname
        printf "\n"
        # configname="config-svr"
        CONFIG_ARRAY=()
        for ((i=1;i<=${confignum};i++))
        do
            read -p "請輸入MongoDB config server第 ${i} 個的位置 (IP or Host): " cnf_ip_1
            # printf "\n"
            read -p "請輸入MongoDB config server第 ${i} 個的Port (default 27017): " cnf_port_1
            if [ -z "${cnf_port_1}" ]; then
                cnf_port_1="27017"
            fi
            printf "${cnf_ip_1}:${cnf_port_1}\n"
            CONFIG_ARRAY+=("${cnf_ip_1}:${cnf_port_1}")
        done

        # cnfstr="var cfg = {\"_id\": \"${RS}\",\"configsvr\": true,\"members\": ["
        # INIT_CMD="rs.initiate(cfg, { force: true }); \\n rs.reconfig(cfg, { force: true }); \\n rs.status();"
        cnfstr=""
        for ((i=0; i<${#CONFIG_ARRAY[@]}; i++))
        do
            num=$((i+100))
            cnfstr="${cnfstr} {\"_id\": ${num},\"host\": \"${CONFIG_ARRAY[i]}\"},"
        done
        cnfstr="${cnfstr::-1}"
        # cnfstr="${cnfstr::-1}]}; \\n ${INIT_CMD}"
        echo "config: ${cnfstr}"
    else
        Notify ongoing "Copy mongod.conf file..."
        cp ./mongod-config.conf /etc/mongod.conf
        #### Add ssl config parameter
        SSL_MongoDB_Config /etc/mongod.conf
    fi
fi

# shard
if [[ $mtype == "3" ]]; then
    read -p "請輸入欲建立第幾個MongoDB shard (1 or 2...): " shardnum
    printf "\n"
    read -p "請輸入欲建立幾個MongoDB ReplicaSet 成員(1 or 2...): " rsnum
    printf "\n"
    read -p "請輸入欲建立第幾個的MongoDB ReplicaSet 成員(1 or 2...): " shardrsnum
    printf "\n"
    if [[ $shardrsnum == "1" ]]; then
        shardname="shard-${shardnum}"

        RS_ARRAY=()
        for ((i=1;i<=${rsnum};i++))
        do
            read -p "請輸入MongoDB rs server第 ${i} 個的位置 (IP or Host): " rs1
            # printf "\n"
            read -p "請輸入MongoDB rs server第 ${i} 個的Port (default 27017): " rsp1
            if [ -z "${rsp1}" ]; then
                rsp1="27017"
            fi
            printf "${rs1}:${rsp1}\n"
            RS_ARRAY+=("${rs1}:${rsp1}")
        done
        rsstr=""
        for ((i=0; i<${#RS_ARRAY[@]}; i++))
        do
            num=$((i+100))
            rsstr="${rsstr} {\"_id\": ${num},\"host\": \"${RS_ARRAY[i]}\"},"
        done
        rsstr="${rsstr::-1}"
        echo "rs: ${rsstr}"
    else
        Notify ongoing "Copy mongod.conf file..."
        cp ./mongod-shard.conf ./mongod-shard-config.conf
        sed -i 's/replSetName: shard-1/replSetName: shard-'"${shardnum}"'/g' mongod-shard-config.conf
        #### Add ssl config parameter
        SSL_MongoDB_Config mongod-shard-config.conf
        cp ./mongod-shard-config.conf /etc/mongod.conf
    fi
fi

# Copy CA File
cat CA/certificate.crt CA/private.key > /etc/ssl/mongodb.pem
cp CA/ca_bundle.crt /etc/ssl/mongodbCA_bundle.crt

Install_MongoDB

if [[ $mtype != "1" ]]; then
    Notify ongoing "Reload & Enable MongoDB Service..."
    systemctl daemon-reload
    systemctl enable mongod
fi

# set keyfile
mkdir -p /var/lib/mongo
if [[ $envtype == "1" ]]; then
    cp ./mongo-keyfile /var/lib/mongo/mongo-keyfile
else
    cp ./mongo-keyfile-prod /var/lib/mongo/mongo-keyfile
fi
chmod 400 /var/lib/mongo/mongo-keyfile
chown mongod:mongod /var/lib/mongo/mongo-keyfile
# for CentOS chcon permission
# chcon system_u:object_r:mongod_var_lib_t:s0 /var/lib/mongo/mongo-keyfile

if [[ $shardrsnum != "1" ]]; then
    systemctl start mongod
fi

# systemctl status mongod
# sleep 5

# Setup Mongos Server
if [[ $mtype == "1" ]]; then
    systemctl stop mongod
    sleep 3
    systemctl disable mongod

    # Full Time Diagnostic Data Capture (FTDC) DB自動蒐集診斷資訊用
    # 因monogs有可能沒建立此資料夾導致禁用此功能, 因此預先建立, 基本上這些資訊是給監控使用
    mkdir -p /var/log/mongodb/mongos.diagnostic.data
    chmod -R 700 /var/log/mongodb/mongos.diagnostic.data
    chown -R mongod:mongod /var/log/mongodb/mongos.diagnostic.data

    cp ./mongos.service /usr/lib/systemd/system/mongos.service
    systemctl daemon-reload
    systemctl enable mongos

    # mongod-router.conf
    rm /etc/mongos.conf
    rm ./mongod-router-config.conf
    cp ./mongod-router.conf ./mongod-router-config.conf
    sed -i 's/configDB: config-svr/configDB: '"${cnfstr}"'/g' mongod-router-config.conf

    #### Add ssl config parameter
    SSL_MongoDB_Config mongod-router-config.conf

    mv /etc/mongos.conf /etc/mongos.conf.bak
    # mv ./mongod-router-config.conf /etc/mongos.conf
    # chown -R mongod:mongod /etc/mongos.conf
    echo "$(< ./mongod-router-config.conf)" > /etc/mongos.conf
    rm ./mongod-router-config.conf

    sleep 1
    systemctl start mongos

    Notify ongoing "Waiting for startup..."
    until mongo --eval 'quit(db.runCommand({ ping: 1 }).ok ? 0 : 2)' ${SSL_PARAMETER} &>/dev/null; do
        printf '.'
        sleep 1
    done

    Notify ongoing "Create user..."
    echo "use admin; db.createUser( {user: \"${username}\", pwd: \"${password}\", roles: [{role: \"root\", db: \"admin\"}]} );"
    mongo --eval "use admin \\n db.createUser( {user: \"${username}\", pwd: \"${password}\", roles: [{role: \"root\", db: \"admin\"}]} );\\n" ${SSL_PARAMETER}
    #     mongo ${SSL_PARAMETER} <<EOF
    # use admin
    # db.createUser( {user: "${username}", pwd: "${password}", roles: [{role: "root", db: "admin"}]} );
    # EOF

    EVAL=""
    for shardstr in "${SHARD_ARRAY[@]}"
    do
        EVAL="${EVAL} sh.addShard(\"${shardstr}\");"
    done
    EVAL="${EVAL} sh.status();"
    echo "${EVAL}"
    mongo -u "${username}" -p "${password}" --eval "${EVAL}" ${SSL_PARAMETER}

fi

# Setup config server 1
if [[ $mtype == "2" && $cnfnum == "1" ]]; then
    Notify ongoing "Setup config on rs1..."
    # mongod --port 27017 --dbpath /var/lib/mongo &
    systemctl start mongod

    # echo "use admin; db.createUser( {user: \"${username}\", pwd: \"${password}\", roles: [{role: \"root\", db: \"admin\"}]} );"
    # mongo --eval "use admin; \\n db.createUser( {user: \"${username}\", pwd: \"${password}\", roles: [{role: \"root\", db: \"admin\"}]} );\\n"
    # mongo --eval 'use admin \n db.createUser( {user: "'${username}'", pwd: "'${password}'", roles: [{role: "root", db: "admin"}]} );'
    mongo <<EOF
use admin
db.createUser( {user: "${username}", pwd: "${password}", roles: [{role: "root", db: "admin"}]} );
EOF

    # mongod --shutdown
    systemctl stop mongod
    sleep 3
    Notify ongoing "Copy mongod.conf file..."
    # cp ./mongod-config.conf /etc/mongod.conf
    cp ./mongod-config.conf ./mongod-config-config.conf
    sed -i 's/replSetName: config-svr/replSetName: '"${configname}"'/g' mongod-config-config.conf
    mv /etc/mongod.conf /etc/mongod.conf.bak
    # mv ./mongod-config-config.conf /etc/mongod.conf

    #### Add ssl config parameter
    SSL_MongoDB_Config mongod-config-config.conf

    echo "$(< ./mongod-config-config.conf)" > /etc/mongod.conf
    rm ./mongod-config-config.conf
    # chown -R mongod:mongod /etc/mongod.conf

    sleep 1
    systemctl start mongod

    Notify ongoing "Waiting for startup..."
    until mongo ${SSL_PARAMETER} --eval 'quit(db.runCommand({ ping: 1 }).ok ? 0 : 2)' &>/dev/null; do
        printf '.'
        sleep 1
    done

    Notify ongoing "Started.."
    Notify ongoing "Setup config rs..."

    # EVAL="${cnfstr}"
    # mongo ${SSL_PARAMETER} -u "${username}" -p "${password}" --eval "${EVAL}"

    mongo ${SSL_PARAMETER} -u "${username}" -p "${password}" <<EOF
var cfg = {"_id": "${configname}","configsvr": true,"members": [${cnfstr}]};
rs.initiate(cfg, { force: true });
rs.reconfig(cfg, { force: true });
rs.status();
EOF
fi

# Setup shard server
if [[ $mtype == "3" && $shardrsnum == "1" ]]; then
    Notify ongoing "Setup shard on rs1"
    # mongod --port 27017 --dbpath /var/lib/mongo &
    systemctl start mongod
    sleep 3

    Notify ongoing "Create user..."
    # mongo --eval "use admin \n db.createUser( {user: ${username}, pwd: ${password}, roles: [{role: \"root\", db: \"admin\"}]} );"
    mongo <<EOF
use admin
db.createUser( {user: "${username}", pwd: "${password}", roles: [{role: "root", db: "admin"}]} );
EOF

    # mongod --shutdown
    systemctl stop mongod
    sleep 3

    Notify ongoing "Copy mongod.conf file..."
    # rm ./mongod-shard-config.conf
    cp ./mongod-shard.conf ./mongod-shard-config.conf
    sed -i 's/replSetName: shard-1/replSetName: shard-'"${shardnum}"'/g' mongod-shard-config.conf
    mv /etc/mongod.conf /etc/mongod.conf.bak
    # mv ./mongod-shard-config.conf /etc/mongod.conf

    #### Add ssl config parameter
    SSL_MongoDB_Config mongod-shard-config.conf

    echo "$(< ./mongod-shard-config.conf)" > /etc/mongod.conf
    rm ./mongod-shard-config.conf
    # chown -R mongod:mongod /etc/mongod.conf
    # rm /tmp/mongodb-27017.sock
    # cp ./mongod-shard.conf /etc/mongod.conf
    sleep 1
    systemctl start mongod
    # sleep 3

    Notify ongoing "Waiting for startup.."
    until mongo ${SSL_PARAMETER} --eval 'quit(db.runCommand({ ping: 1 }).ok ? 0 : 2)' &>/dev/null; do
        printf '.'
        sleep 1
    done

    Notify ongoing "Started.."
    Notify ongoing "Setup shard rs..."

    mongo ${SSL_PARAMETER} -u "${username}" -p "${password}" <<EOF
var cfg = {"_id": "${shardname}","protocolVersion": 1,"members": [${rsstr}]};
rs.initiate(cfg, { force: true });
rs.reconfig(cfg, { force: true });
rs.status();
EOF
fi

Notify success "End setup mongodb."
```

## Uninstall MongoDB Cluster (purge all)

```bash
#!/usr/bin/env bash

echo "Stop & Remove Mongos"
systemctl stop mongos
sleep 3
systemctl disable mongos
rm /usr/lib/systemd/system/mongos.service
rm /lib/systemd/system/mongos.service
systemctl daemon-reload
rm /etc/mongos.conf

echo ""
echo "Stop & Remove MongoDB Package"
service mongod stop
sleep 3
if [ "$1" == '-u' ]; then
    apt-get purge mongodb-org* -y
else
    yum erase $(rpm -qa | grep mongodb-org) -y
fi
rm -r /var/log/mongodb
rm -r /var/lib/mongo
rm /etc/mongod.conf
rm /tmp/mongodb-27017.sock
rm /var/lib/mongo/mongo-keyfile
rm /var/lib/mongo/mongo-keyfile-prod
sed -i 's/exclude=mongodb-org,mongodb-org-server,mongodb-org-shell,mongodb-org-mongos,mongodb-org-tools//g' /etc/yum.conf

echo ""
echo "Remove MongoDB CA File"
rm /etc/ssl/mongodb.pem
rm /etc/ssl/mongodbCA_bundle.crt
```

## mongod config file

```
# mongod.conf

# for documentation of all options, see:
#   http://docs.mongodb.org/manual/reference/configuration-options/

# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log
  # verbosity: 1

# Where and how to store data.
storage:
  dbPath: /var/lib/mongo
  journal:
    enabled: true

#  engine:
#  mmapv1:
#  wiredTiger:

# how the process runs
processManagement:
  fork: true  # fork and run in background
  pidFilePath: /var/run/mongodb/mongod.pid  # location of pidfile
  timeZoneInfo: /usr/share/zoneinfo

# network interfaces
net:
  port: 27017
  bindIp: 0.0.0.0

security:
  authorization: enabled
  keyFile: /var/lib/mongo/mongo-keyfile

#operationProfiling:

replication:
  replSetName: config-svr

sharding:
  clusterRole: configsvr

## Enterprise-Only Options:

#auditLog:

#snmp:

```

## mongod shard config

```
# mongod.conf

# for documentation of all options, see:
#   http://docs.mongodb.org/manual/reference/configuration-options/

# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log
  # verbosity: 1

# Where and how to store data.
storage:
  dbPath: /var/lib/mongo
  journal:
    enabled: true

#  engine:
#  mmapv1:
#  wiredTiger:

# how the process runs
processManagement:
  fork: true  # fork and run in background
  pidFilePath: /var/run/mongodb/mongod.pid  # location of pidfile
  timeZoneInfo: /usr/share/zoneinfo

# network interfaces
net:
  port: 27017
  bindIp: 0.0.0.0

security:
  authorization: enabled
  keyFile: /var/lib/mongo/mongo-keyfile

#operationProfiling:

replication:
  replSetName: shard-1

sharding:
  clusterRole: shardsvr

## Enterprise-Only Options:

#auditLog:

#snmp:

```

## mongod router config

```
# mongod.conf

# for documentation of all options, see:
#   http://docs.mongodb.org/manual/reference/configuration-options/

# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongos.log
  # verbosity: 1

# Where and how to store data.
# storage:
#   dbPath: /var/lib/mongo
#   journal:
#     enabled: true

#  engine:
#  mmapv1:
#  wiredTiger:

# how the process runs
processManagement:
  fork: true  # fork and run in background
  pidFilePath: /var/run/mongodb/mongos.pid  # location of pidfile
  timeZoneInfo: /usr/share/zoneinfo

# network interfaces
net:
  port: 27017
  bindIp: 0.0.0.0

security:
  keyFile: /var/lib/mongo/mongo-keyfile

#operationProfiling:

# replication:
#   replSetName: shard-1

sharding:
  configDB: config-svr

## Enterprise-Only Options:

#auditLog:

#snmp:

```

## mongos.service file

```
[Unit]
Description=MongoDB Database Server
Documentation=https://docs.mongodb.org/manual
After=network.target

[Service]
User=mongod
Group=mongod
#Environment="OPTIONS=-f /etc/mongos.conf"
#EnvironmentFile=-/etc/sysconfig/mongos
#ExecStart=/usr/bin/mongos $OPTIONS
ExecStart=/usr/bin/mongos --config /etc/mongos.conf
ExecStartPre=/usr/bin/mkdir -p /var/run/mongodb
ExecStartPre=/usr/bin/chown mongod:mongod /var/run/mongodb
ExecStartPre=/usr/bin/chmod 0755 /var/run/mongodb
PermissionsStartOnly=true
PIDFile=/var/run/mongodb/mongos.pid
Type=forking
#Type=simple
# file size
LimitFSIZE=infinity
# cpu time
LimitCPU=infinity
# virtual memory size
LimitAS=infinity
# open files
LimitNOFILE=64000
# processes/threads
LimitNPROC=64000
# locked memory
LimitMEMLOCK=infinity
# total threads (user+kernel)
TasksMax=infinity
TasksAccounting=false
# Recommended limits for for mongod as specified in
# http://docs.mongodb.org/manual/reference/ulimit/#recommended-settings

[Install]
WantedBy=multi-user.target

```
