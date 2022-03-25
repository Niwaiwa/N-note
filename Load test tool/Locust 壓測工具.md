###### tags: `壓測`

# Locust 壓測工具

Python 異步壓測框架, 比Jmeter更輕量, 能在Local機器上執行且記憶體消耗量極少  
壓測腳本自由度高, 可自由撰寫

## 壓測基本說明

壓測時要注意腳本設計, 避免做了測試卻都不是實際可用的結果

### DDOS負載方式

因為Locust是異步併發的壓測工具, 所以需要注意腳本設計  
如果一個task裡面有很多個request, 沒任何延時的情況會在1秒內盡可能地做完  
這就可能讓測試變成DDOS的測試, 也就是測試瞬間高併發讓服務瞬間死亡的測試

### 通常負載方式

如果通常的使用複數的task且task內只有一個request, 那就是每秒都只發一個request, 這樣比較符合壓測時的每秒併發的測試方式  
如果只用一個task要測試連續的操作, 要記得設定延時(大概設定為1s), 這樣才會符合每秒併發的測試

## 使用方式

```
docker-compose up
```

### 複數worker

```
docker-compose up --scale worker=2
```

### 多台機器分佈式執行

修改 docker compose file 的 master-host, 指定master ip 之後分別啟動master及worker

```
docker-compose up master

docker-compose up --scale worker=2 worker
```

## WEB UI 介面

```
http://host:8089/
```

## 設定檔說明

有一些設定會因為在腳本內使用了特定的設定而被覆蓋, 最外層command 參數會蓋掉設定檔的參數

參數          | 說明
--------------|:-----
locustfile = locust_file.py  | # 執行哪個壓測腳本
headless = false  | # 是否不開啟web UI介面
master = true  | # 是否是master, 會被command參數蓋掉
expect-workers = 5  | # 期望的worker, 實際沒在用
host = https://host  | # 指定打壓測的目標host
users = 100  | # 預定的併發數量, 可不設定, 使用壓測腳本的設定
run-time = 60s  | # 預訂的執行時間
spawn-rate = 10  | # 預定增加的併發數量/每秒
web-host = 0.0.0.0  | # web UI server listen 的 host
web-port = 8089  | # web UI server listen 的 port
csv = APItest  | # 是否要在執行完後匯出結果
tags = all  | # 指定那些tag的task要執行壓測

## 範例說明

每個task被當作一個要執行的任務執行  
tag會根據執行時指定的tag執行指定的task  
wait_time是每次task要隔多久才執行一次, 預設1s  
self.client繼承自HttpUser的功能, 為request session  
  
※詳細可參考 官方文件 或 Github 取得更詳細的範例  


```python
class TestAPI(HttpUser):
    wait_time = between(1, 1)  # 每次task隔多久才執行

    @tag('normal')
    @task
    def index(self):
        self.client.get("/")  # 發出http request, 並且每個client發出的request都會加入計數
        self.client.cookies.clear()  # 是否每次呼叫都要清掉cahe
```

## 資料來源

[https://docs.locust.io/en/stable/index.html]

## docker-compose file

```
version: '3'

services:
  master:
    image: locustio/locust
    ports:
      - "8089:8089"
      - "5557:5557"
    volumes:
      - ./:/mnt/locust
    command: -f /mnt/locust/locust_file.py --master --config=/mnt/locust/master.conf
  
  worker:
    image: locustio/locust
    volumes:
      - ./:/mnt/locust
    command: -f /mnt/locust/locust_file.py --worker --master-host 10.1.30.19 --config=/mnt/locust/master.conf

```

## 參照

```
import time
import logging
import locust.stats

from API.PlatformAPI import PlatformAPI
from locust import HttpUser, task, tag, LoadTestShape


class StagesShape(LoadTestShape):
    """
    A simply load test shape class that has different user and spawn_rate at
    different stages.
    Keyword arguments:
        stages -- A list of dicts, each representing a stage with the following keys:
            duration -- When this many seconds pass the test is advanced to the next stage
            users -- Total user count
            spawn_rate -- Number of users to start/stop per second
            stop -- A boolean that can stop that test at a specific stage
        stop_at_end -- Can be set to stop once all stages have run.
    """

    # stages = [
    #     {"duration": 150, "users": 100, "spawn_rate": 10},
    #     {"duration": 300, "users": 200, "spawn_rate": 10},
    #     {"duration": 450, "users": 300, "spawn_rate": 10},
    #     {"duration": 600, "users": 400, "spawn_rate": 10},
    #     {"duration": 750, "users": 500, "spawn_rate": 10},
    #     {"duration": 900, "users": 600, "spawn_rate": 10},
    #     {"duration": 1050, "users": 700, "spawn_rate": 10},
    #     {"duration": 1200, "users": 800, "spawn_rate": 10},
    #     {"duration": 1350, "users": 900, "spawn_rate": 10},
    #     {"duration": 1500, "users": 1000, "spawn_rate": 10},
    #     {"duration": 1650, "users": 1100, "spawn_rate": 10},
    #     {"duration": 1800, "users": 1200, "spawn_rate": 10},
    #     {"duration": 1950, "users": 1300, "spawn_rate": 10},
    #     {"duration": 2100, "users": 1400, "spawn_rate": 10},
    #     {"duration": 2250, "users": 1500, "spawn_rate": 10},
    #     {"duration": 2400, "users": 1600, "spawn_rate": 10},
    #     {"duration": 2550, "users": 1700, "spawn_rate": 10},
    #     {"duration": 2700, "users": 1800, "spawn_rate": 10},
    #     {"duration": 2850, "users": 900, "spawn_rate": 10},
    #     {"duration": 3000, "users": 0, "spawn_rate": 10},
    # ]

    # stages = [
    #     {"duration": 150, "users": 200, "spawn_rate": 10},
    #     {"duration": 300, "users": 400, "spawn_rate": 10},
    #     {"duration": 450, "users": 600, "spawn_rate": 10},
    #     {"duration": 600, "users": 800, "spawn_rate": 10},
    #     {"duration": 750, "users": 900, "spawn_rate": 10},
    #     {"duration": 900, "users": 1000, "spawn_rate": 10},
    #     {"duration": 1050, "users": 1100, "spawn_rate": 10},
    #     {"duration": 1200, "users": 1200, "spawn_rate": 10},
    #     {"duration": 1350, "users": 1300, "spawn_rate": 10},
    #     {"duration": 1500, "users": 1400, "spawn_rate": 10},
    #     {"duration": 1650, "users": 1500, "spawn_rate": 10},
    #     {"duration": 1800, "users": 1600, "spawn_rate": 10},
    #     {"duration": 1950, "users": 1700, "spawn_rate": 10},
    #     {"duration": 2100, "users": 1800, "spawn_rate": 10},
    #     {"duration": 2250, "users": 1900, "spawn_rate": 10},
    #     {"duration": 2400, "users": 2000, "spawn_rate": 10},
    #     {"duration": 2460, "users": 1, "spawn_rate": 40},
    # ]

    # stages = [
    #     {"duration": 150, "users": 100, "spawn_rate": 10},
    #     {"duration": 300, "users": 200, "spawn_rate": 10},
    #     {"duration": 450, "users": 300, "spawn_rate": 10},
    #     {"duration": 600, "users": 400, "spawn_rate": 10},
    #     {"duration": 750, "users": 500, "spawn_rate": 10},
    #     {"duration": 900, "users": 600, "spawn_rate": 10},
    #     {"duration": 960, "users": 1, "spawn_rate": 10},
    #     {"duration": 220, "users": 30, "spawn_rate": 10},
    #     {"duration": 230, "users": 10, "spawn_rate": 10},
    #     {"duration": 660, "users": 1, "spawn_rate": 1},
    # ]

    # stages = [
    #     {"duration": 150, "users": 50, "spawn_rate": 10},
    #     {"duration": 300, "users": 100, "spawn_rate": 10},
    #     {"duration": 450, "users": 150, "spawn_rate": 10},
    #     {"duration": 600, "users": 200, "spawn_rate": 10},
    #     {"duration": 750, "users": 250, "spawn_rate": 10},
    #     {"duration": 900, "users": 300, "spawn_rate": 10},
    #     {"duration": 940, "users": 1, "spawn_rate": 10},
    # ]

    stages = [
        {"duration": 1800, "users": 100, "spawn_rate": 10},
    ]

    def tick(self):
        run_time = self.get_run_time()

        for stage in self.stages:
            if run_time < stage["duration"]:
                tick_data = (stage["users"], stage["spawn_rate"])
                return tick_data

        return None


class TestAPI(HttpUser):
    logger = logging.getLogger('TestAPI')
    # wait_time = between(1, 1)

    @tag('normal')
    @task
    def index(self):
        # self.logger.info('index')
        self.client.get("/")
        # self.client.cookies.clear()

    @tag('all')
    @task
    def all(self):
        pAPI = PlatformAPI()

        url, headers, data = pAPI.create_acct()
        self.client.post(url, json=data, headers=headers)
        time.sleep(1)
        
        url, headers, data, group_name = pAPI.acct_balance()
        self.client.get(url, headers=headers, name=group_name)
        time.sleep(1)

    # @task(3)
    # def view_item(self):
    #     for item_id in range(10):
    #         self.client.get(f"/item?id={item_id}", name="/item")
    #         time.sleep(1)

    def on_start(self):
        pass
        # self.logger.info('start')
        # self.client.get("/", json={"username":"foo", "password":"bar"})
        # self.client.post("/login", json={"username":"foo", "password":"bar"})

```