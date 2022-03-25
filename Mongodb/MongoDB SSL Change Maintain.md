###### tags: `MongoDB`
# MongoDB SSL Change Maintain

## 更換SSL CA憑證步驟

```
1. 確認API中心PYTHON服務已關閉
2. 關閉Cluster步驟
3. COPY 新憑證到指定位置
4. 開啟Cluster步驟
5. 確認服務是否可連上DB
```

### COPY 新憑證到指定位置 (已事先準備好憑證)

如果目標位置有檔案, 就輸入 y 覆蓋
理論上ssh登入就會在該帳號的home目錄底下(/home/alan 或 /home/hao)

```
cp mongodb.pem /etc/ssl/mongodb.pem
cp mongodbCA_bundle.crt /etc/ssl/mongodbCA_bundle.crt
```

#### 準備憑證的方式

根據從MIS那邊取得的憑證通常有三種檔案 CA_bundle、CA、Key 需透過下面方式合併再放到指定位置才行

```
cat CA/certificate.crt CA/private.key > /etc/ssl/mongodb.pem
cp CA/ca_bundle.crt /etc/ssl/mongodbCA_bundle.crt
```

### 關閉Cluster步驟

```
1. Disable the Balancer(在router執行)
    1. sh.stopBalancer()
2. Stop mongos routers
    1. systemctl stop mongos.service
3. Stop each shard replica set (先從secondary開始關)
    1. systemctl stop mongod.service
4. Stop config servers (先從secondary開始關)
    1. systemctl stop mongod.service
```

### 開啟Cluster步驟

```
1. Start config servers & Check status
    1. systemctl start mongod.service
    2. rs.status()
2. Start each shard replica set & Check status
    1. systemctl start mongod.service
    2. rs.status()
3. Start mongos routers
    1. systemctl start mongos.service
    2. sh.status()
4. Re-Enable the Balancer & Check status(在router執行)
    1. sh.startBalancer()
    2. sh.status()
```
