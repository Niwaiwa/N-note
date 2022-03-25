###### tags: `EFK`

# fluentd configure

fluentd各種設定及方法

## fluentd dockerfile

從fluentd設定ilm需要elasticsearch-xpack  
要使用官方非預設的http library(typhoeus)的話, 需要libcurl

```Dockerifle
FROM fluent/fluentd:v1.12
USER root
RUN apk add --update --virtual .build-deps \
    sudo build-base ruby-dev \
                    # tzdata \
    # && cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    # && apk add libcurl \  # for typhoeus dependancy
    && sudo gem install fluent-plugin-elasticsearch -v 4.3.3 \
    && sudo gem install elasticsearch-xpack -v 7.11.1 \
    # && sudo gem install typhoeus -v 1.4.0 \
    && sudo gem sources --clear-all \
    && apk del .build-deps \
    && rm -rf /var/cache/apk/* \
    /home/fluent/.gem/ruby/2.3.0/cache/*.gem
    
COPY fluentd/fluent_20211228.conf /fluentd/etc/fluent.conf
# COPY fluentd/template/* /fluentd/etc/template/

ENTRYPOINT ["fluentd", "-c", "/fluentd/etc/fluent.conf", "-p", "/fluentd/plugins"]
```

## 防止重試時出現重複的資料

參照Generate Hash ID

```cnf
<filter **>
  @type elasticsearch_genid
  hash_id_key _hash    # storing generated hash id key (default is _hash)
</filter>
<match **>
  @type elasticsearch
  id_key _hash # specify same key name which is specified in hash_id_key
  remove_keys _hash # Elasticsearch doesn't like keys that start with _
  # other settings are omitted.
</match>
```

## 參數解釋

deflector_alias 為index template的index.lifecycle.rollover_alias用

## 抱怨不足

1. 沒參數可以單純附加alias
2. 沒參數可以自動附加六位數字(000001), 不過附加了大概會導致不會跟著rollover的index走, 所以使用rollover寫入index時還是要指定alias或手動創建


## 參考建立指令

```
#!/usr/bin/env bash
#

# ========== Variable information. ============

# ==================== END ====================

BuildImages(){
    docker build -t efk_fluentd -f Dockerfile .
}

PushImages(){
    docker tag efk_fluentd:latest 10.1.1.102:5000/efk_fluentd:latest
    docker push 10.1.1.102:5000/efk_fluentd:latest
}

SwarmDeploy(){
    if [[ $(docker network ls --filter name=els | grep els) ]]; then echo 1; else docker network create --driver=overlay --attachable els; fi;
    docker stack deploy -c docker-compose4-fluentd.yml efk
}

SwarmUpdate(){
    docker service update -q --force --image 10.1.1.102:5000/efk_fluentd:latest efk_fluentd
}

SwarmScale(){
    docker service scale efk_fluentd=$SCALE_NUM
}

SwarmRemove(){
    docker service rm efk_fluentd
}

helps(){
    echo
}

if [ $1 == 'build' ] then
    BuildImages
elif [ $1 == 'deploy' ] then
    SwarmDeploy
elif [ $1 == 'update' ] then
    SwarmUpdate
elif [ $1 == 'scale' ] then
    SCALE_NUM=$2
    SwarmScale
elif [ $1 == 'remove' ] then
    SwarmRemove
elif [ $1 == '-h' ]
    helps
elif [ $1 == '--help' ] then
    helps
else
    helps
fi

```

## docker compose file

```
version: '3.6'

services:

  fluentd:
    image: efk_fluentd:latest
    ports:
      - "24224:24224"
      - "24224:24224/udp"
      - "24444:24444"
    volumes:
      - ./fluentd/fluent_20211228.conf:/fluentd/etc/fluent.conf
    networks:
      - els
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.labels.efk == efk1
      update_config:
        parallelism: 1
        order: start-first
    stop_grace_period: 1m
    logging:
      driver: "json-file"
      options:
        max-size: "2g"

networks:
  els:
    driver: overlay
    name: els
    external: true

```