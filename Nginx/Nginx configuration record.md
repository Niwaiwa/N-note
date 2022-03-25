###### tags: `Nginx`
# Nginx configuration record

各種使用過的設定的紀錄

## PHP-FPM指定PATH轉發到另一個FPM去

轉發FPM方法

### 只轉發特定PATH到另一個FPM

如果使用set重找DNS, 須加上resolver  
須注意DNS的cache時間, 可以修改valid時間  

```
resolver 127.0.0.11 ipv6=off valid=15s;  # docker 容器內部DNS位置
```

```
location ~ ^/arks/api(.*)$ {
    # 只有/arks/api PATH以後的才轉發
    rewrite ^/arks/api/(.*)$ /$1 break;
    try_files $uri $uri/ /index.php?$query_string;
}

location ~ \.php$ {
    fastcgi_pass ark-api:9000;

    set $new_url $request_uri;
    if ($new_url ~ ^/arks/api(.*)$) {
        # 如果要每次重找DNS則使用set
        # set $arkpro ark-pro;
        # fastcgi_pass $arkpro:9000;

        # set $new_url $1;  # 調整轉發PATH $1或是原始PATH全轉發
        fastcgi_pass ark-pro:9000;
    }
    fastcgi_split_path_info ^(.+\.php)(/.+)$;

    fastcgi_index index.php;
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_param PATH_INFO $fastcgi_path_info;
    fastcgi_param REQUEST_URI $new_url;
}
```

### 只轉發特定PATH到另一個NGINX

如果另一個FPM前面有一個NGINX, 那直接轉發NGINX較簡單  
如果使用set重找DNS, 須加上resolver  
須注意DNS的cache時間, 可以修改valid時間  

```
location ~ ^/arks/api(.*)$ {
    resolver 127.0.0.11 ipv6=off valid=5s;
    set $arkpro http://ark-pro-nginx;
    proxy_pass $arkpro;
}
```

### Load Balance and check upstream

nginx_upstream_check_module

```
https://github.com/yaoweibin/nginx_upstream_check_module
```