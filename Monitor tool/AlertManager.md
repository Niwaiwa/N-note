###### tags: `PMM` `監控`
# AlertManager

使用docker 建立alertmanager, 不過alertmanager只是很單純的警報處理工具, 內部設定幾乎只需要調整設定檔就好  
其他詳細的可以找官方github裡面的example  
如果其他工具本身有支援alertmanager, 就不需要自己額外建立server處理  

## alertmanager.yml

receivers: 轉發警報通知到webhook或其他支援的receivers  
route: 可以設定多層級的群組group  
group_by: 為針對哪個資料欄位對通知資料進行分類  
group_wait: 收到通知後等待幾秒才送出通知, 此時會將收到的通知資料group_by起來  
group_interval: 等待多久才將新收到的通知再次送出  
repeat_interval: 等待多久才重新送出已送出的通知  

```
global:
  resolve_timeout: 5m

route:
  receiver: 'custom-webhook'
  group_by: ['alertname']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 1h


receivers:
- name: 'custom-webhook'
  webhook_configs:
  - url: 'http://10.1.30.19:8080/v1/alert_notify'
```

## compose file

```
version: '3'

services:
  alertmanager:
    image: bitnami/alertmanager:latest
    volumes:
      - ./alertmanager.yml:/opt/bitnami/alertmanager/conf/config.yml
    ports:
      - 9099:9093

```

## PMM 警報通知alertmanager的格式

通常根據警報通知格式, 來選擇要使用的資料  

```
{"receiver": "team-X-webhook", 
"status": "firing", 
"alerts": [
    {"status": "firing", 
    "labels": {
        "agent_id": "/agent_id/ff0c8c5f-8456-460a-800c-610985b4da9a", 
        "agent_type": "mongodb_exporter", 
        "alertgroup": "mongod_replset_state", 
        "alertname": "replsetStateChange", 
        "cluster": "mongo-4.2", 
        "environment": "4.2", 
        "instance": "/agent_id/ff0c8c5f-8456-460a-800c-610985b4da9a", 
        "job": "mongodb_exporter_agent_id_ff0c8c5f-8456-460a-800c-610985b4da9a_hr-5s", 
        "machine_id": "/machine_id/3af0d5181b8c411dbf196706560c937b", 
        "node_id": "/node_id/0ab68383-b65d-4f5c-a1cf-3d1a3b59d089", 
        "node_name": "mongo-test6", 
        "node_type": "generic", 
        "replication_set": "shard-1", 
        "service_id": "/service_id/0bf39047-1b01-41e7-8e2a-3ba09769ab67", 
        "service_name": "mongo-4.2-rs1-01", 
        "service_type": "mongodb", 
        "set": "shard-1", 
        "severity": "page"}, 
    "annotations": {
        "description": "changed", 
        "summary": "ReplSet State changed"}, 
    "startsAt": "2021-02-05T09:04:35.198957921Z", 
    "endsAt": "0001-01-01T00:00:00Z", 
    "generatorURL": "http://localhost:9090/prometheus/api/v1/11031867447847715739/5439709290564009487/status", 
    "fingerprint": "e98c6b0bd87910b9"}
    ], 
"groupLabels": {"alertname": "replsetStateChange"}, 
"commonLabels": {"agent_id": "/agent_id/ff0c8c5f-8456-460a-800c-610985b4da9a", 
    "agent_type": "mongodb_exporter", 
    "alertgroup": "mongod_replset_state", 
    "alertname": "replsetStateChange", 
    "cluster": "mongo-4.2", 
    "environment": "4.2", 
    "instance": "/agent_id/ff0c8c5f-8456-460a-800c-610985b4da9a", 
    "job": "mongodb_exporter_agent_id_ff0c8c5f-8456-460a-800c-610985b4da9a_hr-5s", 
    "machine_id": "/machine_id/3af0d5181b8c411dbf196706560c937b", 
    "node_id": "/node_id/0ab68383-b65d-4f5c-a1cf-3d1a3b59d089", 
    "node_name": "mongo-test6", 
    "node_type": "generic", 
    "replication_set": "shard-1", 
    "service_id": "/service_id/0bf39047-1b01-41e7-8e2a-3ba09769ab67", 
    "service_name": "mongo-4.2-rs1-01", 
    "service_type": "mongodb", 
    "set": "shard-1", 
    "severity": "page"}, 
"commonAnnotations": {"description": "changed", 
    "summary": "ReplSet State changed"}, 
"externalURL": "http://c7560e4774f7:9093", 
"version": "4", 
"groupKey": "{}:{alertname="replsetStateChange"}", 
"truncatedAlerts": 0}
```

## 官方範例

```
{
    "receiver": "team-X-webhook",
    "status": "firing",
    "alerts": [
        {
            "status": "firing",
            "labels": {
                "alertname": "DiskRunningFull",
                "dev": "sda1",
                "instance": "example1"
            },
            "annotations": {
                "info": "The disk sda1 is running full",
                "summary": "please check the instance example1"
            },
            "startsAt": "2021-02-05T05:40:25.1130774Z",
            "endsAt": "0001-01-01T00:00:00Z",
            "generatorURL": "",
            "fingerprint": "8320f0af89d48747"
        },
        {
            "status": "firing",
            "labels": {
                "alertname": "DiskRunningFull",
                "dev": "sda2",
                "instance": "example1"
            },
            "annotations": {
                "info": "The disk sda2 is running full",
                "runbook": "the following link http://test-url should be clickable",
                "summary": "please check the instance example1"
            },
            "startsAt": "2021-02-05T05:40:25.1130774Z",
            "endsAt": "0001-01-01T00:00:00Z",
            "generatorURL": "",
            "fingerprint": "e1d3beb1bb24a40c"
        },
        {
            "status": "firing",
            "labels": {
                "alertname": "DiskRunningFull",
                "dev": "sda1",
                "instance": "example2"
            },
            "annotations": {
                "info": "The disk sda1 is running full",
                "summary": "please check the instance example2"
            },
            "startsAt": "2021-02-05T05:40:25.1130774Z",
            "endsAt": "0001-01-01T00:00:00Z",
            "generatorURL": "",
            "fingerprint": "831ef0af89d40470"
        },
        {
            "status": "firing",
            "labels": {
                "alertname": "DiskRunningFull",
                "dev": "sdb2",
                "instance": "example2"
            },
            "annotations": {
                "info": "The disk sdb2 is running full",
                "summary": "please check the instance example2"
            },
            "startsAt": "2021-02-05T05:40:25.1130774Z",
            "endsAt": "0001-01-01T00:00:00Z",
            "generatorURL": "",
            "fingerprint": "74eed9368316aaba"
        },
        {
            "status": "firing",
            "labels": {
                "alertname": "DiskRunningFull",
                "dev": "sda1",
                "instance": "example3",
                "severity": "critical"
            },
            "annotations": {},
            "startsAt": "2021-02-05T05:40:25.1130774Z",
            "endsAt": "0001-01-01T00:00:00Z",
            "generatorURL": "",
            "fingerprint": "7666d3970f6e15cf"
        },
        {
            "status": "firing",
            "labels": {
                "alertname": "DiskRunningFull",
                "dev": "sda1",
                "instance": "example3",
                "severity": "warning"
            },
            "annotations": {},
            "startsAt": "2021-02-05T05:40:25.1130774Z",
            "endsAt": "0001-01-01T00:00:00Z",
            "generatorURL": "",
            "fingerprint": "6543bc164be4f176"
        }
    ],
    "groupLabels": {
        "alertname": "DiskRunningFull"
    },
    "commonLabels": {
        "alertname": "DiskRunningFull"
    },
    "commonAnnotations": {},
    "externalURL": "http://91a4505567d6:9093",
    "version": "4",
    "groupKey": "{}:{alertname=\"DiskRunningFull\"}",
    "truncatedAlerts": 0
}

```

## python alert server

自定義web 服務, 接收alert manager的通知, 並處理之後送到想送的目標去  
下面是轉換範例  

```
def send_alertmanager_notify(data: dict):
    alert_type = data.get("status", "warning")
    # cusfields = []  # rocketchat custom field
    events = []
    for alert_info in data.get("alerts"):
        event = {
            "alertname": alert_info.get("labels").get("alertname"),
            "service_name": alert_info.get("labels").get("service_name"),
        }
        if alert_info.get("annotations"):
            anno = alert_info['annotations']
            if anno.get("summary"):
                event.update({"summary": anno.get("summary")})
            if anno.get("severity"):
                event.update({"severity": anno.get("severity")})
            if anno.get("description"):
                # description_text_list = anno.get("description").split(" - ")
                # anno["description"] = f"{description_text_list[0]} - {mongo_state.get(int(description_text_list[1]))}"
                event.update({"description": anno.get("description")})
        events.append(event)
    content = {
        "alert_from": f"Alertmanager {env_mode.upper()}",
        "events": events
    }

    # attachments = {
    #     "username": "Alertmanager",
    #     # "text": "Example Prometheus Alert message",
    #     "attachments": [{
    #         # "color": alertColor,
    #         "title": "Prometheus notification",
    #         "fields": cusfields
    #     }]
    # }
    # logger.info(alert_type)
    # logger.info(content)
    return send_message(alert_type, content=content)
```