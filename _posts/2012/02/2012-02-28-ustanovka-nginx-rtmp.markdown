---
layout: post
title: Установка NGINX-RTMP
date: '2012-02-28 11:45:00'
tags:
- ubuntu
- dev
- other
---


В последнее время занимаюсь онлайн вещанием через интернет перепробовал много медиа серверов (FMS, Wowza, Early, etc), но остановился на модуле к веб серверу nginx, под названием <a href="https://github.com/arut/nginx-rtmp-module/" target="_blank">nginx-rtmp</a>. Этот пост посвящен быстрой установке и настройке медиа сервера для живого вещания на чистую Ubuntu 12.04.<br />
Большую часть того что я здесь опишу вы можете найти в стандартной документации на <a href="https://github.com/arut/nginx-rtmp-module/wiki" target="_blank">GitHub</a>. Ну начнем..
<!--more-->

Первое что нужно это поставить все необходимые пакеты

{% highlight bash %}
sudo apt-get install git-core gcc pkg-config make libpcre3 libpcre3-dev libssl-dev
{% endhighlight %}
Далее создадим папку <strong>build</strong>:
{% highlight bash %}
mkdir build
cd ~/build
{% endhighlight %}
Теперь нам нужно скачать и установить <strong>ffmpeg</strong>:
{% highlight bash %}
git clone git://source.ffmpeg.org/ffmpeg.git ffmpeg
cd ffmpeg
sudo ./configure --prefix=/opt/ffmpeg --disable-yasm --enable-shared
sudo make
sudo make install
{% endhighlight %}

Скачиваем <strong>nginx-rtmp-module</strong>:

{% highlight bash %}
cd ~/build
git clone git://github.com/arut/nginx-rtmp-module.git
{% endhighlight %}

Скаиваем и собираем <strong>nginx</strong>:

{% highlight bash %}
wget http://nginx.org/download/nginx-1.2.7.tar.gz
tar xzf nginx-1.2.7.tar.gz
cd nginx-1.2.7
./configure --add-module=~/build/nginx-rtmp-module --add-module=~/build/nginx-rtmp-module/hls --with-cc-opt=-I/opt/ffmpeg/include --with-ld-opt='-L/opt/ffmpeg/lib -Wl,-rpath=/opt/ffmpeg/lib'
make
make install
{% endhighlight %}

Теперь давайте создадим первое приложение для онлайн вещания.
Нам нужно отредактировать файл конфигурации nginx:

{% highlight bash %}
sudo nano /usr/local/nginx/conf/nginx.conf
{% endhighlight %}

И заменим содержимое на следующий код:
{% highlight nginx %}
#user  nobody;
worker_processes  1;
error_log  logs/error.log debug;
events {
   worker_connections  1024;
}
http {
   include       mime.types;
   default_type  application/octet-stream;
   sendfile        on;
   keepalive_timeout  65;
   server {
       listen       8080;
       server_name  localhost;
       # sample handlers
       #location /on_play {
       #    if ($arg_pageUrl ~* localhost) {
       #        return 201;
       #    }
       #    return 202;
       #}
       #location /on_publish {
       #    return 201;
       #}
       #location /vod {
       #    alias /var/myvideos;
       #}
       # rtmp stat
       location /stat {
           rtmp_stat all;
           rtmp_stat_stylesheet stat.xsl;
       }
       location /stat.xsl {
           # you can move stat.xsl to a different location
           root /usr/build/nginx-rtmp-module;
       }
       # rtmp control
       location /control {
           rtmp_control all;
       }
       error_page   500 502 503 504  /50x.html;
       location = /50x.html {
           root   html;
       }
   }
}

rtmp {
   server {
       listen 1935;
       ping 30s;
       notify_method get;
       application myapp {
           live on;
           # sample play/publish handlers
           #on_play http://localhost:8080/on_play;
           #on_publish http://localhost:8080/on_publish;
           # sample recorder
           #recorder rec1 {
           #    record all;
           #    record_interval 30s;
           #    record_path /tmp;
           #    record_unique on;
           #}
           # sample HLS
           #hls on;
           #hls_path /tmp/hls;
           #hls_sync 100ms;
       }
       # Video on demand
       #application vod {
       #    play /var/Videos;
       #}
       # Video on demand over HTTP
       #application vod_http {
       #    play http://localhost:8080/vod/;
       #}
   }
}
{% endhighlight %}
Теперь мы можем транслировать онлайн поток с помощью <strong>Adobe Flash Media Live Encoder</strong>
Нам нужно направить поток на адрес <code>rtmp://YOU-IP/myapp/mystream</code>
Забирать поток вы можете с этого же адреса
