---
layout: post
title: Letsencrypt, NGINX и Docker Compose
date: '2016-06-03 16:33:50'
tags:
- nginx
- docker
---

Привожу самый простой вариант того как запустить letsencrypt с помощью docker compose.

<!--more-->

docker-compose.yml

{% highlight yaml %}
version: '2'

services:
  nginx:
    restart: always
    image: nginx
    hostname: example.com
    build: ./compose/nginx
    ports:
      - 80:80
      - 443:443
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - ./letsencrypt/conf:/etc/letsencrypt
      - ./letsencrypt/html:/tmp/letsencrypt
    environment:
      - LE_RENEW_HOOK=docker kill -s HUP @CONTAINER_NAME@

  letsencrypt:
    restart: always
    image: kvaps/letsencrypt-webroot
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock
      - ./letsencrypt/conf:/etc/letsencrypt
      - ./letsencrypt/html:/tmp/letsencrypt
    links:
      - nginx
    environment:
      - DOMAINS=example.com www.example.com
      - EMAIL=example@gmail.com
      - WEBROOT_PATH=/tmp/letsencrypt
      - EXP_LIMIT=30
      - CHECK_FREQ=30
{% endhighlight %}

Как видно из docker-compose.yml мы используем [kvaps/letsencrypt-webroot](https://hub.docker.com/r/kvaps/letsencrypt-webroot/) для контейнера letsencrypt и расшариваем файловую систему между ним и контейнером nginx. Dockerfile для контейнера nginx находится в директории `/compose/nginx`

Dockerfile для nginx

{% highlight dockerfile %}
FROM nginx:latest
ADD nginx.conf /etc/nginx/nginx.conf
ADD dhparams.pem /etc/nginx/dhparams.pem
{% endhighlight %}

Мы используем стандартную сборку nginx и просто копируем nginx.conf из директории c Dockerfile в контейнер.

Содержимое nginx.conf:

{% highlight nginx %}
user  nginx;
worker_processes  1;
error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;
events {
    worker_connections  1024;
}
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  /var/log/nginx/access.log  main;
    sendfile        on;
    keepalive_timeout  65;
    upstream app {
        server django:5000;
    }
    server {
        listen 80;
        charset     utf-8;
        client_max_body_size 32m;
        location / {
            # checks for static file, if not found proxy to app
            try_files $uri @proxy_to_app;
        }
        location @proxy_to_app {
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $http_host;
            proxy_redirect off;
            proxy_pass   http://app;
        }
        location '/.well-known/acme-challenge' {
            default_type "text/plain";
            root        /tmp/letsencrypt;
        }
    }
    server {
        listen                          443 ssl;
        charset                         utf-8;
        ssl_certificate                 /etc/letsencrypt/live/example.com/fullchain.pem;
        ssl_certificate_key             /etc/letsencrypt/live/example.com/privkey.pem;
        ssl_protocols                   TLSv1 TLSv1.1 TLSv1.2;
        ssl_prefer_server_ciphers       on;
        ssl_dhparam                     /etc/nginx/dhparams.pem;
        ssl_ciphers                     'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
        ssl_session_timeout             1d;
        ssl_session_cache               shared:SSL:50m;
        ssl_stapling                    on;
        ssl_stapling_verify             on;
        add_header                      Strict-Transport-Security max-age=15768000;
        client_max_body_size 32m;
        location / {
            # checks for static file, if not found proxy to app
            try_files $uri @proxy_to_app;
        }
        location @proxy_to_app {
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $http_host;
            proxy_redirect off;
            proxy_pass   http://app;
        }
        location '/.well-known/acme-challenge' {
            default_type "text/plain";
            root        /tmp/letsencrypt;
        }
    }
}
{% endhighlight %}

В nginx.conf мы используем volumes из контейнера letsencrypt чтобы получить доступ к ключу в директории `/.well-known/acme-challenge` и доступ с самим ключам SSL.

Для генерации `dhparams.pem` локально используем команду `sudo openssl dhparam -out ./dhparam.pem 2048`

Собственно это все, поите все должно работать даже при копипасте.
