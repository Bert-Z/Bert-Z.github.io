---
title: OpenStack上nginx端口映射
date: 2020-06-05 15:21:57
tags: [Config,Nginx,Docker]
categories:
cover: /images/nginx/nginx.png
---
<meta name="referrer" content="no-referrer" />

# OpenStack上nginx端口映射

> 唉～讲真的每次用每次查，有点麻烦了，这次还是都记录下来吧。

## docker-run.sh

```
#!/bin/sh

docker run -d --name nginx \
    -p 80:80 \
    --net host \
    -v /home/ubuntu/CA/nginx/nginx.conf:/etc/nginx/nginx.conf \
    nginx

```



## nginx.conf

```
user  root;
worker_processes  1;
 
error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;
 
events {
    worker_connections  1024;
}
 
 
http {
    include       mime.types;
    default_type  application/octet-stream;
 
    sendfile        on;
 
    keepalive_timeout  65;
 
 
    server {
    		# 直接用公网端口
        listen       30271;
        server_name  127.0.0.1:8090;
 
        location / {
            proxy_pass   http://127.0.0.1:8090;
        }
 
    }
 
}
```

