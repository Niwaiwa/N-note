###### tags: `EFK`

# Fluentd 安裝 (td-agent) (mariadb slow query log)

此為在實體機器Linux上安裝fluentd的方式  
若不是centos可自行改用ubuntu作法  

```bash
#/bin/bash

# install centos td-agent
curl -L https://toolbelt.treasuredata.com/sh/install-redhat-td-agent3.sh | sh

# install dependancy package for plugin
yum install -y gcc make autoconf

# install plugin by td-agent-gem
td-agent-gem install fluent-plugin-elasticsearch -v 4.3.3 \
fluent-plugin-kafka \
fluent-plugin-rewrite-tag-filter \
fluent-plugin-multiline-parser \
fluent-plugin-record-modifier \
fluent-plugin-out-http \
fluent-plugin-mysqlslowquery -v 0.0.9 \
elasticsearch-xpack -v 7.9.0

# set custom config 
cp /etc/td-agent/td-agent.conf /etc/td-agent/td-agent.conf.bak
echo '
<system>
    rpc_endpoint 0.0.0.0:24444
</system>

<source>
  @type mysql_slow_query
  path /var/log/mysql/mariadb-slow.log
  pos_file /tmp/mysql/mariadb-slow.pos
  tag "mariadb.slow_query.#{Socket.gethostname.downcase}"
   <parse>
     @type none
   </parse>
</source>
# <source>
#   @type tail
#   format none
#   path /var/log/mysql/general.log
#   pos_file /tmp/mysql/general.pos
#   tag mariadb.general
# </source>
<source>
  @type tail
  format none
  path /var/log/mysql/mariadb.err
  pos_file /tmp/mysql/mariadb.pos
  tag "mariadb.error.#{Socket.gethostname.downcase}"
</source>

<match mariadb**>
  @type copy
  <store>
    @type stdout
  </store>
  <store>
    @type elasticsearch
    host 10.1.1.104
    port 9200
    default_elasticsearch_version 7
    verify_es_version_at_startup true
    suppress_type_name true
    include_tag_key true
    include_timestamp true

    index_name ${tag}

    template_overwrite true
    template_name ""
    template_file /etc/td-agent/mariadb_template.json
    customize_template {"<<TAG>>": "${tag}"}
    rollover_index true

    application_name log
    enable_ilm true
    ilm_policy_id mariadb-default-policy
    ilm_policy { "policy": { "phases": { "hot": { "min_age": "0ms", "actions": { "rollover": { "max_age": "10d", "max_size": "50gb" } } }, "delete": { "min_age": "40d", "actions": { "delete": {} } } } } }
    ilm_policy_overwrite true
    <buffer>
      flush_interval 3s
    </buffer>
  </store>
</match>

<match **>
  @type stdout
</match>
' > /etc/td-agent/td-agent.conf

# set custom template file
echo '
{
    "order": 1500,
    "index_patterns": ["<<TAG>>-log*"], 
    "settings": {
        "number_of_shards": 1,
        "number_of_replicas": 1,
        "index.lifecycle.name": "mariadb-default-policy", 
        "index.lifecycle.rollover_alias": "<<TAG>>-log"
    }
}
' > /etc/td-agent/mariadb_template.json

# mod td-agent user to taget group, because need permisson to read log file
usermod -aG mysql td-agent

# start td-agent service
systemctl start td-agent
```

## mariadb_template

```
{
    "order": 1500,
    "index_patterns": ["<<TAG>>-log*"], 
    "settings": {
        "number_of_shards": 1,
        "number_of_replicas": 1,
        "index.lifecycle.name": "mariadb-default-policy", 
        "index.lifecycle.rollover_alias": "<<TAG>>-log"
    }
}

```