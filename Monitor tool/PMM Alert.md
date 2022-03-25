###### tags: `PMM` `監控`

# PMM Alert

PMM 監控, 對alertmanager的規則設定紀錄  
目前警報採用外部自建的alertmanager  

## 對MongoDB的警報設定紀錄

```
groups:
- name: mongod_replset_state
  rules:
  - alert: replsetStateChange
    expr: mongodb_mongod_replset_my_state{service_type="mongodb"} == 1
    for: 0s
    labels:
      severity: page
    annotations:
      summary: ReplSet State changed
      description: changed
```

```
groups:
- name: Mongo
  rules:
  - alert: Mongo status change
    expr: mongodb_mongod_replset_my_state != mongodb_mongod_replset_my_state offset 1m
    labels:
      severity: Critical
    annotations:
       summary: "DB {{ $labels.service_name }} status change!!!"
       description: "{{ $labels.service_name }} status has changed!!!!!"
```

```
groups:
- name: mongod_replset_state
  rules:

  # Alert for any instance that is unreachable for >5 minutes.
  - alert: replsetStateChange
    expr: mongodb_mongod_replset_my_state != mongodb_mongod_replset_my_state offset 1m
    annotations:
      description: "{{ $labels.service_name }} ReplSet state changed to - {{ $value }}"
```
