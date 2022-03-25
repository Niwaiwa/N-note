###### tags: `MariaDB` 

# MariaDB Master Slave(GTID)

## master先創建slave用的user

```sql
create user if not exists 'replica'@'%' identified by 'password';
GRANT USAGE ON *.* to replica@'%' IDENTIFIED BY 'passowrd';
GRANT ALL PRIVILEGES on *.* TO 'replica'@'%';
FLUSH PRIVILEGES;
```

## 調整master的my.cnf的DB設定 (如果MIS開新的站台已經用新的設定檔就跳過這一步驟)

1. 先備份my.cnf檔案
    cp /etc/mysql/my.cnf /etc/mysql/my.cnf.bak

2. 替換/etc/mysql/my.cnf的檔案
    cp /xxx/my.cnf /etc/mysql/my.cnf

3. 重啟DB服務
    systemctl restart mariadb.service

4. 檢查DB服務
    systemctl status mariadb.service

## 備份master  (for使用GTID 做新slave)(xxx為密碼)

mysqldump -uroot -pxxx --master-data=1 --gtid --all-databases > master_backup.sql

## 準備root密碼跟slave同步用user密碼，並執行scripts安裝10.3.X(10.3.23)的DB跟初始化跟創建user

sudo sh master-slave-gtid.sh

1. 輸入master密碼
2. 輸入slave密碼
3. 輸入2 (因為建立的是slave)

## 還原master備份，xxx為root密碼

mysql -uroot -pxxx < master_backup.sql

## 如果第7點Slave_IO_State的狀態不正常，可以參考以下可能情況進行除錯

一直連線不到master的情況，如果是linode的環境(VM)，可能是因為selinux設定造成DB無法主動對外連線，可參考以下開放對外連線，另一個是假設master是selinux的話，要被slave連線就要開放

```sh
    getsebool -a | grep mysql    #找到mariadb有關的變數
    setsebool -P mysql_connect_any on    #開放mysql對外連線
    setsebool -P selinuxuser_mysql_connect_enabled on  #開放使用者連線到本機端
```

## 進slave確認還原後的slave gtid狀態，確認gtid_slave_pos參數是否跟master的gtid_current_pos一樣(完全停止服務為前提)

1. 確認slave還原後的gtid_slave_pos位置
    SELECT @@GLOBAL.gtid_slave_pos;
2. 確認master當前的gtid_current_pos的位置
    SELECT @@GLOBAL.gtid_current_pos;
3. 確認master跟slave都在同個位置就OK

## 設定slave對master的slave同步設定

1. 重設狀態(清除多餘的master設定)
    RESET MASTER;
2. 設定gtid嚴格模式(如偵測到不連續的gtid複製，就會停止複製)
    SET GLOBAL gtid_strict_mode=ON;
3. 設定slave對master的同步設定，ip為master ip，port為master port，xxx為master同步用帳號的密碼(帳號都固定名稱)，MASTER_USE_GTID基本上都固定
    CHANGE MASTER TO MASTER_HOST='ip', MASTER_PORT=port, MASTER_USER='replica', MASTER_PASSWORD='xxx', MASTER_USE_GTID=current_pos;
4. 都確認設定正常後，啟動SLAVE複製
    START SLAVE;
    ※如果master備份出來的gtid位置跟實際的不一樣，在確定資料都為相同的情況下，可以在執行第(3)點之前用下面指令重設gtid位置
    SET GLOBAL gtid_slave_pos = "0-1-1";

## 檢查啟動複製後的SLAVE複製狀態

如果第一個的Slave_IO_State為"Waiting for master to send event"，那就是正常連線到了master也確認同步到了最新的GTID資料

    SHOW SLAVE STATUS;

### GTID複製問題，清空所有東西，重新建立slave

※失敗要重做的話清空所有東西

```sh
systemctl stop mariadb.service
rm -r /var/lib/mysql
rm /etc/my.cnf.d/*
yum remove -y MariaDB-server
```

## 安裝用腳本

如果要安裝的版本不在官方預設位置, 需要過去版本的話要使用archive位置

```
baseurl = https://archive.mariadb.org//mariadb-10.3.23/yum/centos7-amd64
```

master-slave-gtid sh

```
#/bin/bash
echo -e "MariaDB Master-Master 配置開始"
read -sp "請輸入Mariadb root 密碼: " password
printf "\n"
read -sp "請再次輸入Mariadb root 密碼: " password1
printf "\n\n"
if [[ $password != $password1 ]]; then
	echo '密碼不符合'
	exit 1
fi
read -sp "請輸入Mariadb Replica 使用密碼: " rep_password
printf "\n"
read -sp "請再次輸入Mariadb Replica 使用密碼: " rep_password1
printf "\n\n"
if [[ $rep_password != $rep_password1 ]]; then
	echo 'Replica密碼不符合'
	exit 1
fi
read -p "請輸入MariaDB server_id , 1 or 2 ..., 第一台輸入 1, 第二台輸入 2" node
printf "\n\n"
case "$node" in
[1-9])
echo -e "OK";;
*       )
echo Wrong
exit;;
esac
echo -e "指定 Mariadb repos 來源..."
# 指定Mariadb 安裝repos來源
echo "[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.3.23/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1
" >> /etc/yum.repos.d/mariadb.repo
echo -e "安裝 MariaDB-server & MariaDB-client ..."
# 安裝Mariadb
yum install -y MariaDB-server MariaDB-client
echo -e "配置mysql 設定檔..."
echo "#
# These groups are read by MariaDB server.
# Use it for options that only the server (but not clients) should see
#
# See the examples of server my.cnf files in /usr/share/mysql/
#
# this is read by the standalone daemon and embedded servers
[server]
# this is only for the mysqld standalone daemon
[mysqld]
#
# * Galera-related settings
#
[galera]
# Mandatory settings
#wsrep_on=ON
#wsrep_provider=
#wsrep_cluster_address=
#binlog_format=row
#default_storage_engine=InnoDB
#innodb_autoinc_lock_mode=2
#
# Allow server to accept connections on all interfaces.
#
#bind-address=0.0.0.0
#
# Optional setting
#wsrep_slave_threads=1
#innodb_flush_log_at_trx_commit=0
# this is only for embedded server
[embedded]
# This group is only read by MariaDB servers, not by MySQL.
# If you use the same .cnf file for MySQL and MariaDB,
# you can put MariaDB-only options here
[mariadb]
# This group is only read by MariaDB-10.3 servers.
# If you use the same .cnf file for MariaDB of different versions,
# use this group for options that older servers don't understand
[mariadb-10.3]
port=3306
log-bin
server_id="$node"
log-basename=master"$node"
binlog-format=mixed
log_output=FILE
general_log
general_log_file=general.log
log_disabled_statements='slave,sp'
slow_query_log
slow_query_log_file=slow.log
long_query_time=2.0
log_slow_disabled_statements='admin,call,slave,sp'
log_queries_not_using_indexes=ON
min_examined_row_limit=100000
log_slow_rate_limit=5
log_error=/var/log/mysql/mariadb.err
log_warnings=3
log_slave_updates
expire_logs_days=10
" >> /etc/my.cnf.d/my.cnf
echo -e "Active MariaDB service..."
systemctl enable mariadb.service
systemctl start mariadb.service
echo -e "Mariadb 初始化權限配置中..."
printf "%s\n" "" "y" "$password" "$password" "y" "n" "y" "y" | /usr/bin/mysql_secure_installation
echo -e "Mariadb Replica 權限配置中...\n"
mysql -uroot -p$password -e "DELETE FROM mysql.user WHERE user='';" \
-e "GRANT ALL ON *.* TO 'root'@'%' IDENTIFIED BY '"$password"';" \
-e "GRANT USAGE ON *.* to replica@'%' IDENTIFIED BY '"$rep_password"';" \
-e "GRANT ALL PRIVILEGES on *.* to replica@'%';" \
-e "FLUSH PRIVILEGES;"

```

### 設定檔

```
# MariaDB database server configuration file.
#
# You can copy this file to one of:
# - "/etc/mysql/my.cnf" to set global options,
# - "~/.my.cnf" to set user-specific options.
# 
# One can use all long options that the program supports.
# Run program with --help to get a list of available options and with
# --print-defaults to see which it would actually understand and use.
#
# For explanations see
# http://dev.mysql.com/doc/mysql/en/server-system-variables.html

# This will be passed to all mysql clients
# It has been reported that passwords should be enclosed with ticks/quotes
# escpecially if they contain "#" chars...
# Remember to edit /etc/mysql/debian.cnf when changing the socket location.
[client]
port		= 3306
socket		= /var/run/mysqld/mysqld.sock

# Here is entries for some specific programs
# The following values assume you have at least 32M ram

# This was formally known as [safe_mysqld]. Both versions are currently parsed.
[mysqld_safe]
socket		= /var/run/mysqld/mysqld.sock
nice		= 0
timezone        = 'America/Caracas'
[mysqld]
#
# * Basic Settings
#
user		= mysql
pid-file	= /var/run/mysqld/mysqld.pid
socket		= /var/run/mysqld/mysqld.sock
port		= 3306
basedir		= /usr
datadir		= /var/lib/mysql
tmpdir		= /tmp
lc_messages_dir	= /usr/share/mysql
lc_messages	= en_US
skip-external-locking
#
# Instead of skip-networking the default is now to listen only on
# localhost which is more compatible and is not less secure.
#bind-address		= 127.0.0.1
#
# * Fine Tuning
#
max_connections		= 100
connect_timeout		= 10
wait_timeout		= 28800
max_allowed_packet	= 16M
thread_cache_size       = 128
sort_buffer_size	= 4M
bulk_insert_buffer_size	= 16M
tmp_table_size		= 32M
max_heap_table_size	= 32M
#
# * MyISAM
#
# This replaces the startup script and checks MyISAM tables if needed
# the first time they are touched. On error, make copy and try a repair.
myisam_recover_options = BACKUP
key_buffer_size		= 128M
#open-files-limit	= 2000
table_open_cache	= 400
myisam_sort_buffer_size	= 512M
concurrent_insert	= 2
read_buffer_size	= 2M
read_rnd_buffer_size	= 1M
#
# * Query Cache Configuration
#
# Cache only tiny result sets, so we can fit more in the query cache.
query_cache_limit		= 128K
query_cache_size		= 64M
# for more write intensive setups, set to DEMAND or OFF
#query_cache_type		= DEMAND
#
# * Logging and Replication
#
# Both location gets rotated by the cronjob.
# Be aware that this log type is a performance killer.
# As of 5.1 you can enable the log at runtime!
#general_log_file        = /var/log/mysql/mysql.log
#general_log             = 1
#log_disabled_statements='slave,sp'
#
# Error logging goes to syslog due to /etc/mysql/conf.d/mysqld_safe_syslog.cnf.
#
# we do want to know about network errors and such

log-bin
server_id=1
log-basename=master1
binlog-format=mixed
log_slave_updates

log_output=FILE

log_warnings		= 3
log_error=/var/log/mysql/mariadb.err
#
# Enable the slow query log to see queries with especially long duration
slow_query_log=1
slow_query_log_file	= /var/log/mysql/mariadb-slow.log
long_query_time = 2.0
log_slow_disabled_statements='admin,call,slave,sp'
log_queries_not_using_indexes=ON
min_examined_row_limit=100000
log_slow_rate_limit=5
#log_slow_rate_limit	= 1000
#log_slow_verbosity	= query_plan

#log-queries-not-using-indexes
#log_slow_admin_statements
#
# The following can be used as easy to replay backup logs or for replication.
# note: if you are setting up a replication slave, see README.Debian about
#       other settings you may need to change.
#server-id		= 1
#report_host		= master1
#auto_increment_increment = 2
#auto_increment_offset	= 1
#log_bin			= /var/log/mysql/mariadb-bin
#log_bin_index		= /var/log/mysql/mariadb-bin.index
# not fab for performance, but safer
#sync_binlog		= 1
expire_logs_days	= 10
max_binlog_size         = 100M
# slaves
#relay_log		= /var/log/mysql/relay-bin
#relay_log_index	= /var/log/mysql/relay-bin.index
#relay_log_info_file	= /var/log/mysql/relay-bin.info
#log_slave_updates
#read_only
#
# If applications support it, this stricter sql_mode prevents some
# mistakes like inserting invalid dates etc.
#sql_mode		= NO_ENGINE_SUBSTITUTION,TRADITIONAL
#
# * InnoDB
#
# InnoDB is enabled by default with a 10MB datafile in /var/lib/mysql/.
# Read the manual for more InnoDB related options. There are many!
default_storage_engine	= InnoDB
innodb_buffer_pool_size	= 256M
innodb_log_buffer_size	= 8M
innodb_file_per_table	= 1
innodb_open_files	= 400
innodb_io_capacity	= 400
innodb_flush_method	= O_DIRECT
#
# * Security Features
#
# Read the manual, too, if you want chroot!
# chroot = /var/lib/mysql/
#
# For generating SSL certificates I recommend the OpenSSL GUI "tinyca".
#
# ssl-ca=/etc/mysql/cacert.pem
# ssl-cert=/etc/mysql/server-cert.pem
# ssl-key=/etc/mysql/server-key.pem

#
# * Galera-related settings
#
[galera]
# Mandatory settings
#wsrep_on=ON
#wsrep_provider=
#wsrep_cluster_address=
#binlog_format=row
#default_storage_engine=InnoDB
#innodb_autoinc_lock_mode=2
#
# Allow server to accept connections on all interfaces.
#
#bind-address=0.0.0.0
#
# Optional setting
#wsrep_slave_threads=1
#innodb_flush_log_at_trx_commit=0

[mysqldump]
quick
quote-names
max_allowed_packet	= 16M

[mysql]
#no-auto-rehash	# faster start of mysql but no tab completion

[isamchk]
key_buffer		= 16M

#
# * IMPORTANT: Additional settings that can override those from this file!
#   The files must end with '.cnf', otherwise they'll be ignored.
#
!include /etc/mysql/mariadb.cnf
!includedir /etc/mysql/conf.d/

```