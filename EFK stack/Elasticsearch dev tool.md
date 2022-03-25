###### tags: `EFK` 

# Elasticsearch dev tool

記錄曾使用過或常用的DEV API呼叫方式

# History record

```
GET _search
{
  "query": {
    "match_all": {}
  }
}

POST /_index_template/_simulate/test

GET fluentd*/_ilm/explain
POST fluentd-20210204_ilm/retry

GET _ilm/status
POST _ilm/stop
POST _ilm/start

GET _search
{
  "query": {
    "match_all": {}
  }
}

GET _nodes/stats
GET _cluster/stats
GET _cluster/state
GET _cluster/health
GET _cluster/settings
GET _cluster/settings?include_defaults=true&flat_settings=true
GET _stats
GET _ilm/status
GET _transform/_stats
GET _settings
GET _license/basic_status
GET _tasks
GET _cat/health
GET _cat/nodeattrs
GET _cat/indices

PUT _cluster/settings
{
  "persistent" : {
    "cluster.max_shards_per_node" : "4000"
  }
}
```

## 調整cluster各node可以有多少shard

調整各node的shard數量, 但index數量太多會有另外的讀取速度的問題, 且非最佳實踐, 官方預設為1000  
最好搭配上ilm rollover來控制index數量

```
GET _cluster/settings?include_defaults=true&flat_settings=true

PUT _cluster/settings
{
  "persistent" : {
    "cluster.max_shards_per_node" : "4000"
  }
}
```

## move index state

遷移index state需要有詳細且正確的state參數, 不然無法遷移

```
POST _ilm/move/fluentd-20210204
{
  "current_step": { 
    "phase": "hot",
    "action": "unfollow",
    "name": "wait-for-follow-shard-tasks"
  },
  "next_step": { 
    "phase": "hot",
    "action": "rollover",
    "name": "check-rollover-ready"
  }
}

POST _ilm/move/fluentd-20210204
{
  "current_step": { 
    "phase": "hot",
    "action": "rollover",
    "name": "check-rollover-ready"
  },
  "next_step": { 
    "phase": "hot",
    "action": "complete",
    "name": "complete"
  }
}

POST _ilm/move/fluentd-20210204
{
  "current_step": { 
    "phase": "hot",
    "action": "rollover",
    "name": "check-rollover-ready"
  },
  "next_step": { 
    "phase": "hot",
    "action": "rollover",
    "name": "check-rollover-ready"
  }
}
```

## put new ilm policy

```
PUT _ilm/policy/test
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_size": "20kb",
            "max_age": "1m",
            "max_docs": 10
          }
        }
      }
    }
  }
}
```

## put new index template

```
PUT _index_template/test
{
  "priority": 500,
  "template": {
    "settings": {
      "index": {
        "lifecycle": {
          "name": "test",
          "rollover_alias": "fluentd"
        }
      }
    },
    "aliases": {
      "fluentd-all": {
        "is_write_index": true
      }
    }
  },
  "index_patterns": [
    "fluentd-*"
  ]
}
```

## 預先 put index for ilm

```
PUT fluentd-000001
{
  "aliases": {
    "fluentd-all": { "is_write_index": true } 
  }
}
```

## 手動rollover

可不帶參數直接rollover, 此功能可以方便測試ilm是否設定正確

```
POST /nginx-web-test-alias/_rollover
{
  "conditions": {
    "max_age":   "1m",
    "max_docs":  20,
    "max_size": "20kb"
  }
}
```
