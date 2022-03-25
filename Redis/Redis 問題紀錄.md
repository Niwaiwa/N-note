###### tags: `Redis` `問題記錄`

# Redis 問題紀錄

## 機器異常關機, 重開機後遇到需修復AOF檔案的問題(使用bitnami的image的情況)

使用AOF持久化, 但遇到機器異常關機導致無法正常啟動redis  
此時依照指示使用 redis-check-aof 工具進行修復  
但因為是容器的關係, 有可能不好修復, 此時可以使用別的compose file開啟別的容器, 並且voloume檔案進去再進行修復

```
/var/lib/docker/volumes/ark-redis_redis_data/_data  # voloume的外部位置
/bitnami/redis/data  # redis aof 檔案的內部位置
/opt/bitnami/redis/bin/redis-check-aof --fix /bitnami/redis/data  # 依照指示使用指定的修復工具修復AOF檔案
```

```
docker-compose -f docker-compose-redis2.yml exec ./redis-check-aof --fix /bitnami/redis/data  
```

