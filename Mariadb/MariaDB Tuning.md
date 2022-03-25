###### tags: `MariaDB` `Tuning`

# MariaDB Tuning 相關

## 內部記憶體使用量, 為RDS的預設設定, 可用做其他自建DB的參考

查看mysql記憶體相關各項配置  

```
select @@key_buffer_size / (1024 * 1024)         as key_buffer_size,
       @@query_cache_size / (1024 * 1024)        as query_cache_size,
       @@tmp_table_size / (1024 * 1024)          as tmp_table_size,
       @@innodb_buffer_pool_size / (1024 * 1024) as innodb_buffer_pool_size,
       @@innodb_log_buffer_size / (1024 * 1024)  as innodb_log_buffer_size,
       @@max_connections                         as max_connections,
       @@sort_buffer_size / (1024 * 1024)        as sort_buffer_size,
       @@read_buffer_size / (1024 * 1024)        as read_buffer_size,
       @@read_rnd_buffer_size / (1024 * 1024)    as read_rnd_buffer_size,
       @@join_buffer_size / (1024 * 1024)        as join_buffer_size,
       @@thread_stack / (1024 * 1024)            as thread_stack,
       @@binlog_cache_size / (1024 * 1024)       as binlog_cache_size;
```

計算最大耗用記憶體的量  

```
select ( @@key_buffer_size + @@query_cache_size + @@tmp_table_size + @@innodb_buffer_pool_size + 
         @@max_connections *
         ( @@sort_buffer_size + @@read_buffer_size + @@read_rnd_buffer_size +  @@join_buffer_size + @@thread_stack + @@binlog_cache_size)
           ) / (1024 * 1024) as total_memory_mb
```

查看目前DB狀態  
    
    SHOW ENGINE INNODB STATUS;

查看

```
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 50518294528     -> 分配給innodb的總內存大小, 單位 byte
Dictionary memory allocated 496944         ->  分配給innodb數據字典的內存大小, 單位 byte
Buffer pool size   3014656     ->   innodb buffer pool 的大小,單位page, 此值 * page_size 就是 innodb pool size的大小(單位kb); 
   查看page size指令: select @@innodb_page_size
Free buffers       9731    -> innodb buffer pool 中,閒置的page數量
Database pages     2822495   -> innodb buffer pool 中非閒置的page數量
Old database pages 1041744  -> old 子表中的page數量
Modified db pages  9967  ->  當前buffer pool中被修改的page數量
Percent of dirty pages(LRU & free pages): 0.352
Max dirty pages percent: 75.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 980866, not young 3817746
0.82 youngs/s, 0.12 non-youngs/s
Pages read 2674750, created 190413, written 3626419
0.18 reads/s, 0.29 creates/s, 28.00 writes/s
Buffer pool hit rate 999 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 2822495, unzip_LRU len: 0
I/O sum[7064]:cur[1624], unzip sum[0]:cur[0]
```

## 經常出現 MySQL server has gone away 或 Too many connections

跟連線有關的錯誤，需查看DB的最大連線數是否曾經達到最大

```
show variables like '%Connection_errors_max_connections'; -> 由於達到max_connections限制而被拒絕的連接數
show variables like '%Connection_errors_internal'; -> 由於內部服務器錯誤（例如內存不足錯誤或線程啟動失敗）而被拒絕的連接數
show variables like '%Connection_errors_peer_address'; -> 搜索連接客戶端 IP 地址時的錯誤數
show variables like '%max_connections'; -> 最大連線數
show status like '%Max_used_connections'; -> 服務啟動以來曾經同時使用的最大連線數
SHOW STATUS LIKE 'Threads_connected'; -> 當前連線數
SET GLOBAL max_connections=1024; -> 動態設定最大連線數(或靜態設定 mysql.conf)
```

## 如果MySQL 內存不足

參考

```
https://www.percona.com/blog/2018/06/28/what-to-do-when-mysql-runs-out-of-memory-troubleshooting-guide/
```

## 如果需使用到convert_tz之類的DB內時區轉換功能

需匯入時區檔案才能正常使用時區轉換功能  
如果自行從系統安裝mariadb需要執行指令匯入時區檔案, 如果使用docker image則已經自動匯入完成了

```
mysql_tzinfo_to_sql /usr/share/zoneinfo | mysql -u root mysql
```
測試用指令
```
SELECT CONVERT_TZ(STR_TO_DATE("2021-07-28 01:46:54", "%Y-%m-%d %H:%i:%s" ), 'America/Caracas', 'Asia/Shanghai')
```

參考連結
[mysql_tzinfo_to_sql](https://mariadb.com/kb/en/mysql_tzinfo_to_sql/)
[convert_tz](https://mariadb.com/kb/en/convert_tz/)

## 寫入大量資料遇到 SQLSTATE[HY000]: General error: 1390 Prepared statement contains too many placeholders

DB的placeholders上限為65535(64K), 計算方式為寫入的行數乘上Table的列數, 超過就會無法寫入, 此時應使用chunk寫入, 有可能chunk數量設定太高結果還是觸發錯誤

```
3087行 * 20列 = 61740  # 未超過
3356行 * 20列 = 67120  # 超過, 報錯1390
```

## 使用Percona tool的pt-query-digest工具, 對mariadb slow query做進一步分析

```
cat mariadb-slow.log | docker run -i --rm matsuu/pt-query-digest > analyzed-slow.log
```

因使用docker可以簡單自動化的步驟

```
1. 就找一台機器
2. 每天早上6點半~7點之間 SSH遠端到正式區DB機器COPY前一天的LOG
3. 抓下來之後用docker 指定分析並輸出分析後的檔案
4. 寫個小工具或者shell 腳本 將原始slow log跟分析後的檔案 上傳到火箭
```

## 初始編碼 跟 trigger 關聯的問題

當建立database時的編碼跟table使用的編碼有不同的情況下(初始:utf8_general_ci, table: utf8_unicode_ci), 有可能導致出現以下錯誤訊息, 儘管測試時正常還是有機會觸發, 原因也可能跟trigger有關

* https://stackoverflow.com/a/45192368

```
SQLSTATE[HY000]: General error: 1267 Illegal mix of collations (utf8_general_ci,IMPLICIT) and (utf8_unicode_ci,IMPLICIT) for operation '<>' (SQL: insert into `letter` 
```

### 再現方式 (mariadb 10.3.23)

1. 建立database 使用 COLLATE 'utf8_general_ci';
2. 建立table 使用 COLLATE 'utf8_unicode_ci';
3. 建立trigger
4. INSERT資料
5. 當trigger存在'<>'欄位比較的邏輯就會出現錯誤

### 解決方法

1. 將database的編碼更改為utf8_unicode_ci, 然後將有問題的table重建一次並轉移資料即可正常
2. 將整個database重建並重新匯入資料徹底解決所有問題

## 各種統計數據 Statistics

該資料說明了直接從DB內取得各種統計資料的SQL語法, 比較有用的是index row讀取跟全表row讀取的比較

* https://vettabase.com/blog/understanding-tables-usage-with-user-statistics-percona-server-mariadb/

Rows read with an index vs. read with a tablescan:
```
SELECT
t.TABLE_SCHEMA, t.TABLE_NAME,
t.ROWS_READ AS TOTAL_ROWS_READ,
SUM(i.ROWS_READ) AS INDEX_ROWS_READ,
t.ROWS_READ - SUM(i.ROWS_READ) AS TABLESCAN_ROWS_READ
FROM information_schema.TABLE_STATISTICS t
LEFT JOIN information_schema.INDEX_STATISTICS i
ON t.TABLE_SCHEMA = i.TABLE_SCHEMA AND t.TABLE_NAME = i.TABLE_NAME
WHERE t.TABLE_SCHEMA NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
GROUP BY t.TABLE_SCHEMA, t.TABLE_NAME, t.ROWS_READ
;
```

Find unused tables (since last server restart or User Statistics activation):

```
SELECT
t.TABLE_SCHEMA, t.TABLE_NAME, t.ENGINE
FROM information_schema.TABLES t
LEFT JOIN information_schema.TABLE_STATISTICS s
ON t.TABLE_SCHEMA = s.TABLE_SCHEMA AND t.TABLE_NAME = s.TABLE_NAME
WHERE t.TABLE_SCHEMA NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
AND ROWS_READ IS NULL AND ROWS_CHANGED IS NULL
;
```

## MySQLTuner

MySQL效能分析工具, 會對DB進行分析並提供參考建議

實際使用記憶體參考值
```
Total buffers: 46.4G global + 107.5M per thread (1000 max threads)
```

* https://github.com/major/MySQLTuner-perl
