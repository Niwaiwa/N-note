###### tags: `Python`

# Web Server

筆記

## uwsgi

### install

pip install uwsgi

### uwsgi.ini

```
[uwsgi]
# Http access port.
# If this option comes into effect, we can visit our web site on http://[our IP]:[Port]
# http=:5005

# Uwsgi's ip and port when it is loaded by Nginx
socket=:5000

# Point to the main directory of the Web Site
chdir=/myapp

# Python startup file
wsgi-file=/myapp/index.py

# The application variable of Python Flask Core Oject
callable=app

# Write the uwsgi log
# req-logger = file:/myapp/logs/reqlog.log

# The maximum numbers of Processes
processes=9

# The maximum numbers of Threads
# threads=2

# Uwsgi 狀態管理
stats=127.0.0.1:9191
safe-pidfile=/tmp/uwsgi.pid

die-on-term = true
vacuum = true
master = true

# Uwsgi gevent asynchronous/non-blocking modes, threads option not work with gevent
gevent = 100
limit-as = 2048
listen = 100
reload-on-as = 256
reload-on-rss = 192
max-requests = 2000
gevent-early-monkey-patch = true
#wsgi-env-behaviour = holy
lazy-apps = true

# log 相關
#set-placeholder = log_dir=/myapp/logs
#set-placeholder = log_prefix=uwsgi-whale-
#set-placeholder = log_num=14
#pidfile = /var/run/uwsgi-myservice.pid
#logto = %(log_dir)/%(log_prefix)@(exec://date +%%Y-%%m-%%d).log
#log-reopen = true
## 每日輪換 0點1分
#unique-cron = 1 0 -1 -1 -1 { sleep 66 && kill -HUP $(cat %(pidfile)) && ls -tp %(log_dir)/%(log_prefix)* | grep -v '/$' | tail -n +%(log_num) | xargs -d '\n' -r rm --; } &

```

#### 過去的問題紀錄(先放上來)

##### 全部應加入下面三項設定

此外使用gevent時，uwsgi 的threads option不會work，但應會work在gevent以外
使用gevent的時候，不應該使用threads參數，會導致沒啟用gevent核心

```
gevent = 100
gevent-early-monkey-patch = true
wsgi-env-behaviour = holy
```

##### 依照環境會有問題

    wsgi-env-behaviour = holy
    
##### 要加入這項才會取代內部lib成為異步非阻塞

    gevent-early-monkey-patch = true

##### 無法正常啟動運用的情況可以使用以下參數

目的是切換app啟動方式, 讓app可以使用乾淨的方式啟動, 而不是透過thread啟動導致問題
詳細參照官網說明

    lazy-apps = true


## gunicorn

### install

pip install gunicorn gunicorn[gevent]

### gunicorn_conf.py

```
# Gunicorn config variables
loglevel = "info"
errorlog = "-"  # stderr
accesslog = "-"  # stdout
worker_tmp_dir = "/dev/shm"
graceful_timeout = 30
timeout = 120
keepalive = 5
# threads = 3
workers = 2
max_requests = 2000

# gevent
worker_class = "gevent"
worker_connections = 1000
```