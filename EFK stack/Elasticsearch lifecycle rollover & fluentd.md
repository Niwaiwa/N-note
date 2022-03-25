###### tags: `EFK`

# Elasticsearch lifecycle rollover & fluentd

rollover是透過alias來搜尋對應的index name的aliasm 之後再執行rollover  
對es lifecycle來說, 重點在alias, lifecycle使用alias作為rollover的依據, 而alias對應上的index只能是同一類的index, 這樣rollover才會成立  

## Automate rollover

如果要使用index template來自動rollover index就必須設定index.lifecycle.rollover_alias  
lifecycle在rollover index的時候會搜尋index_patterns對應上的所有index的alias  
之後執行rollover, 但是要注意對應上的所有index必須只有"一個"index的狀態為"is_write_index"為true  
這樣es才會知道哪一個index是要當前最新的index  
換個說法rollover是用下面步驟執行rollover

1. 先建立新的index
2. 設定新的index的is_write_index為true  
3. 之後將舊的index的is_write_index更新為false  
4. 最後再將alias對應到的當前index指到is_write_index為true的新的index  

`https://www.elastic.co/guide/en/elasticsearch/reference/7.10/getting-started-index-lifecycle-management.html#getting-started-index-lifecycle-management`

### rollover必須條件

1. index自帶六位數字(000001)
2. 必須有alias

## kibana 手動設定從fluentd來的全部index

1. 建立index, 
2. 為每個類型(index_pattern)的index建立index_template, 


## 手動設定rollover

```
{
  "index": {
    "lifecycle": {
      "name": "logs",
      "rollover_alias": "{index}"
    }
  }
}
```

## 手動建立index with aliases

```
{
  "aliases": {
    "logs": { 
      "is_write_index": true 
    } 
  }
}
```

## index rollover生命週期, 手動預建立

需手動在index name上加上六位數字(000001)給ilm rollover遷移index使用  

```es
PUT _ilm/policy/fluentd
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

PUT _index_template/fluentd
{
  "priority": 500,
  "template": {
    "settings": {
      "index": {
        "lifecycle": {
          "name": "fluentd",
          "rollover_alias": "fluentd"
        }
      }
    }
  },
  "index_patterns": [
    "fluentd-*"
  ]
}

PUT fluentd-000001
{
  "aliases": {
    "fluentd": { "is_write_index": true } 
  }
}
```

fluentd

```
  @type elasticsearch
  host 192.168.1.101
  port 9200
  default_elasticsearch_version 7
  verify_es_version_at_startup true
  suppress_type_name true

  include_tag_key true

  index_name fluentd
  include_tag_key true
  tag_key @log_name
  <buffer tag>
    flush_interval 1s
    chunk_limit_records 1
  </buffer>
```

## fluentd 仿logstash格式(但失敗)

```
<store>
    @type elasticsearch
    host elasticsearch
    port 9200
    default_elasticsearch_version 7
    verify_es_version_at_startup true
    suppress_type_name true
    # log_es_400_reason true

    include_tag_key true
    # tag_key "tag"
    include_timestamp true

    # 有BUG 不可跟ilm一起使用
    # logstash_format true
    # logstash_prefix tests-${tag}
    # logstash_prefix_separator .
    # logstash_dateformat ""

    index_name tests  # 不使用logstash_format時, 要使用ilm的話要有
    # time_key '60s'

    template_overwrite true
    template_name tests-${tag}
    template_file /fluentd/etc/tests_template.json
    # customize_template {"string_1": "subs_value_1", "string_2": "subs_value_2"}  # 須設定template_name, template_file才有效, 也可以只設定前兩個
    customize_template {"TAG": "${tag}"}
    rollover_index true  # 須設定customize_template才有效, 新版不可與enable_ilm一起設定, 同時有ilm時也不需要此設定
    deflector_alias tests-${tag}  # 須設定rollover_index才有效, 新版不可與enable_ilm一起設定, 同時有ilm時也不需要此設定

    # index_date_pattern "now/d" # defaults to "now/d", 須設定enable_ilm才有效, 且會自動生效, now/w{xxxx.ww}
    application_name ${tag} # defaults to "default", 須設定enable_ilm才有效, 且會自動生效
    # enable_ilm true # Default value is false 
    # ilm_policy_id tests-policy # Default value is logstash-policy
    # ilm_policy { "policy": { "phases": { "hot": { "min_age": "0ms", "actions": { "rollover": { "max_age": "1m", "max_docs": 20, "max_size": "20kb" } } }, "delete": { "min_age": "1h", "actions": { "delete": {} } } } } }
    # ilm_policy_overwrite false

    <buffer tag>
      flush_interval 1s
      chunk_limit_records 1
    </buffer>
  </store>
```

## fluentd ilm

template & fluentd ilm  
使用fluentd plugin來自動生成template & ilm  
但每份設定只能指定一個index

```
{
  "index_patterns": ["test-log-*"], 
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1,
    "index.lifecycle.name": "test-policy", 
    "index.lifecycle.rollover_alias": "test-log"
  }
}
```

```
  <store>
    @type elasticsearch
    host elasticsearch
    port 9200
    default_elasticsearch_version 7
    verify_es_version_at_startup true
    suppress_type_name true
    include_tag_key true

    index_name test

    template_name test_template
    template_file /fluentd/etc/test_template.json
    # customize_template {"string_1": "subs_value_1", "string_2": "subs_value_2"}  # 須設定template_name, template_file才有效, 也可以只設定前兩個
    # rollover_index true  # 須設定customize_template才有效
    # deflector_alias test-log  # 須設定rollover_index才有效

    # index_date_pattern "now/w{xxxx.ww}" # defaults to "now/d", 須設定enable_ilm才有效, 且會自動生效
    application_name log # defaults to "default", 須設定enable_ilm才有效, 且會自動生效
    enable_ilm true # Default value is false 
    ilm_policy_id test-policy # Default value is logstash-policy
    ilm_policy { "policy": { "phases": { "hot": { "min_age": "0ms", "actions": { "rollover": { "max_age": "1m", "max_docs": 20, "max_size": "20kb" } } }, "delete": { "min_age": "1h", "actions": { "delete": {} } } } } }

    <buffer tag>
      flush_interval 1s
      chunk_limit_records 1
    </buffer>
  </store>
```

## 不可用設定1

此設定會出現兩個index, 但只會寫入index_name的index, 同時看不到ilm設定
猜測是像logstash_format的設定

```
    @type elasticsearch
    host elasticsearch
    port 9200
    default_elasticsearch_version 7
    verify_es_version_at_startup true
    suppress_type_name true

    include_tag_key true
    include_timestamp true
    index_name tests

    template_overwrite true
    template_name tests-${tag}
    template_file /fluentd/etc/tests_template.json
    customize_template {"TAG": "${tag}"}  # 須設定template_name, template_file才有效, 也可以只設定前兩個
    rollover_index true  # 須設定customize_template才有效, 新版不可與enable_ilm一起設定, 同時有ilm時也不需要此設定
    deflector_alias tests-${tag}  # 須設定rollover_index才有效, 新版不可與enable_ilm一起設定, 同時有ilm時也不需要此設定

    application_name ${tag} # defaults to "default", 須設定enable_ilm才有效, 且會自動生效

    <buffer tag>
      flush_interval 1s
      chunk_limit_records 1
    </buffer>
  </store>
  ```

## 不可用設定2

此設定會出現alias無法對應的錯誤, 儘管調整成看起來一模一樣, 也還是會出錯

```
<store>
    @type elasticsearch
    host elasticsearch
    port 9200
    default_elasticsearch_version 7
    verify_es_version_at_startup true
    suppress_type_name true
    # log_es_400_reason true

    include_tag_key true
    # tag_key "tag"
    # include_timestamp true

    # 有BUG 不可跟ilm一起使用
    # logstash_format true
    # logstash_prefix ${tag}
    # logstash_prefix_separator .
    # logstash_dateformat ""

    index_name ${tag}  # 不使用logstash_format時, 要使用ilm的話要有
    # time_key '60s'

    rollover_index true  # 須設定customize_template才有效, 新版不可與enable_ilm一起設定, 同時有ilm時也不需要此設定
    deflector_alias ${tag}-alias  # 須設定rollover_index才有效, 新版不可與enable_ilm一起設定, 同時有ilm時也不需要此設定
    template_overwrite true
    template_name ${tag}
    template_file /fluentd/etc/testss_template.json
    # customize_template {"string_1": "subs_value_1", "string_2": "subs_value_2"}  # 須設定template_name, template_file才有效, 也可以只設定前兩個
    customize_template {"{TAG}": "${tag}"}


    <buffer tag>
      flush_interval 1s
      chunk_limit_records 1
    </buffer>
  </store>
```

## ilm fluentd 環境變數設定方式

fluentd 使用env帶入index name

```
{
  "index_patterns": ["mock"],
  "template": {
    "settings": {
      "index": {
        "lifecycle": {
          "name": "mock",
          "rollover_alias": "mock"
        },
        "number_of_shards": "<<shard>>",
        "number_of_replicas": "<<replica>>"
      }
    }
  }
}
```

```
<source>
  @type http
  port 5004
  bind 0.0.0.0
  body_size_limit 32m
  keepalive_timeout 10s
  <parse>
    @type json
  </parse>
</source>

<match kubernetes.var.log.containers.**etl-webserver**.log>
    @type elasticsearch
    @id out_es_etl_webserver
    @log_level info
    include_tag_key true
    host $HOST
    port $PORT
    path "#{ENV['FLUENT_ELASTICSEARCH_PATH']}"
    request_timeout "#{ENV['FLUENT_ELASTICSEARCH_REQUEST_TIMEOUT'] || '30s'}"
    scheme "#{ENV['FLUENT_ELASTICSEARCH_SCHEME'] || 'http'}"
    ssl_verify "#{ENV['FLUENT_ELASTICSEARCH_SSL_VERIFY'] || 'true'}"
    ssl_version "#{ENV['FLUENT_ELASTICSEARCH_SSL_VERSION'] || 'TLSv1'}"
    reload_connections "#{ENV['FLUENT_ELASTICSEARCH_RELOAD_CONNECTIONS'] || 'false'}"   
    reconnect_on_error "#{ENV['FLUENT_ELASTICSEARCH_RECONNECT_ON_ERROR'] || 'true'}"
    reload_on_failure "#{ENV['FLUENT_ELASTICSEARCH_RELOAD_ON_FAILURE'] || 'true'}"
    log_es_400_reason "#{ENV['FLUENT_ELASTICSEARCH_LOG_ES_400_REASON'] || 'false'}"
    logstash_prefix "#{ENV['FLUENT_ELASTICSEARCH_LOGSTASH_PREFIX'] || 'etl-webserver'}"
    logstash_format "#{ENV['FLUENT_ELASTICSEARCH_LOGSTASH_FORMAT'] || 'false'}"
    index_name "#{ENV['FLUENT_ELASTICSEARCH_LOGSTASH_INDEX_NAME'] || 'etl-webserver'}"
    type_name "#{ENV['FLUENT_ELASTICSEARCH_LOGSTASH_TYPE_NAME'] || 'fluentd'}"
    time_key "#{ENV['FLUENT_ELASTICSEARCH_TIME_KEY'] || '@timestamp'}"
    include_timestamp "#{ENV['FLUENT_ELASTICSEARCH_INCLUDE_TIMESTAMP'] || 'true'}"

    # ILM Settings - WITH ROLLOVER support
    # https://github.com/uken/fluent-plugin-elasticsearch#enable-index-lifecycle-management
    rollover_index true
    application_name "etl-webserver"
    index_date_pattern ""
    # Policy configurations
    enable_ilm true
    ilm_policy_id etl-webserver
    ilm_policy_overwrite true
    ilm_policy {"policy": {"phases": {"hot": {"min_age": "0ms","actions": {"rollover": {"max_age": "5m","max_size": "3gb"},"set_priority": {"priority": 100}}},"delete": {"min_age": "30d","actions": {"delete": {"delete_searchable_snapshot": true}}}}}}
    use_legacy_template false
    template_name etl-webserver
    template_file /configs/index-template.json
    template_overwrite true
    customize_template {"<<shard>>": "3","<<replica>>": "0"}

    
    <buffer>
        flush_thread_count "#{ENV['FLUENT_ELASTICSEARCH_BUFFER_FLUSH_THREAD_COUNT'] || '8'}"
        flush_interval "#{ENV['FLUENT_ELASTICSEARCH_BUFFER_FLUSH_INTERVAL'] || '5s'}"
        chunk_limit_size "#{ENV['FLUENT_ELASTICSEARCH_BUFFER_CHUNK_LIMIT_SIZE'] || '8MB'}"
        total_limit_size "#{ENV['FLUENT_ELASTICSEARCH_TOTAL_LIMIT_SIZE'] || '450MB'}"
        queue_limit_length "#{ENV['FLUENT_ELASTICSEARCH_BUFFER_QUEUE_LIMIT_LENGTH'] || '32'}"
        retry_max_interval "#{ENV['FLUENT_ELASTICSEARCH_BUFFER_RETRY_MAX_INTERVAL'] || '60s'}"
        retry_forever false
    </buffer>
</match>
```

## 從fluentd自動設定ilm跟index_template

使用placeholders來自動設定每個index name, index template設定每個index name, index template
可以使用logstash_format或index_name兩種, 但logstash格式各種名稱會帶有額外的字串

index template
```
{
  "index_patterns": ["<<TAG>>-log*"], 
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1,
    "index.lifecycle.name": "test-policy", 
    "index.lifecycle.rollover_alias": "<<TAG>>-log"
  }
}
```

fluentd config
```
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>

<match {nginx**,httpd**}>
    @type elasticsearch
    host 192.168.1.101
    port 9200
    default_elasticsearch_version 7
    verify_es_version_at_startup true
    suppress_type_name true
    include_tag_key true
    include_timestamp true

    # logstash_format true
    # logstash_prefix ${tag}
    # logstash_dateformat ""

    index_name ${tag}

    template_overwrite true
    template_name ""
    template_file /fluentd/etc/test_template.json
    customize_template {"<<TAG>>": "${tag}"}
    rollover_index true

    application_name log
    enable_ilm true
    ilm_policy_id test-policy
    ilm_policy { "policy": { "phases": { "hot": { "min_age": "0ms", "actions": { "rollover": { "max_age": "1m", "max_docs": 20, "max_size": "20kb" } } }, "delete": { "min_age": "1h", "actions": { "delete": {} } } } } }
    ilm_policy_overwrite true

    flush_interval 3s
</match>
```

## 手動rollover

nginx-web為alias也是rollover-target  

* https://www.elastic.co/guide/en/elasticsearch/reference/master/indices-rollover-index.html

```http
POST /nginx-web/_rollover
{
  "conditions": {
    "max_age":   "1m",
    "max_docs":  20,
    "max_size": "20kb"
  }
}

```
