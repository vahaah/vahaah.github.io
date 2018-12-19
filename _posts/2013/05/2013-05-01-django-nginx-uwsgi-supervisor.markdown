---
layout: post
title: Django + nginx + uWSGI + supervisor
date: '2013-05-01 12:09:00'
tags:
- ubuntu
- dev
- django
---

Думаю не нужно объяснять что пользователь под которым вы зашли не должен именоваться root. Ну если с этим проблем не замечено давайте начнем, для начала давайте поставим nginx:
<!--more-->
{% highlight bash %}
sudo apt-get install python-software-properties
sudo add-apt-repository ppa:nginx/stable
sudo apt-get update
sudo apt-get install nginx
{% endhighlight %}

Теперь нам нужно установить `pip`

{% highlight bash %}
sudo apt-get install python-pip</pre>
{% endhighlight %}

И собственно установка нашего виртуального окружения

{% highlight bash %}
sudo apt-get install python-virtualenv</pre>
{% endhighlight %}

Теперь давайте определимся со структурой нашего сферического проекта в вакууме

{% highlight bash %}
/ - за корень возьмем папку нашего проекта
-app
--app2
--app3
--static
--media
--settings.py
--urls.py
--wsgi.py
-manage.py
-env
-logs
-pids
-uwsgi.ini
{% endhighlight %}

По строению дерева как мне кажется все понятно, давайте приступим в первую очередь нужно создать виртуальное окружение.

{% highlight bash %}
virtualenv --prompt="project" ./env
{% endhighlight %}

теперь мы можем активировать наше виртуальное окружение

{% highlight bash %}
source ~/project/env/bin/activate
{% endhighlight %}

<small class="mutted">(Для деактивации нужное ввести deactivate)</small>

Теперь мы можем поставить необходимые пакеты, для начала нам хватит 2-х

{% highlight bash %}
pip install django
pip install uwsgi
{% endhighlight %}

Теперь давайте создадим конфигурационный файл uwsgi.ini

{% highlight bash %}
nano ~/project/uwsgi.ini
{% endhighlight %}

<p>Содержимое файла</p>

{% highlight bash %}
[uwsgi]
home=/home/project/env
chdir=/home/project
master=True
disable-logging=True
vacuum=True
pidfile=/home/project/pids/project.pid
max-requests=5000
socket=127.0.0.1:49001
processes=2

pythonpath=/home/project
env=DJANGO_SETTINGS_MODULE=app.settings
module = django.core.handlers.wsgi:WSGIHandler()
touch-reload=/tmp/project.txt
{% endhighlight %}

Теперь давайте создадим конфиг для supervisor, он позволит нам управлять проектом и вообще будет следить, чтобы все работало

{% highlight bash %}
sudo nano /etc/supervisor/conf.d/project.conf
{% endhighlight %}

И добавляем туда следующий код

{% highlight bash %}
[program:project]
command=/home/project/env/bin/uwsgi /home/project/uwsgi.ini
stdout_logfile=/home/project/logs/wsgi.log
stderr_logfile=/home/project/logs/wsgi_err.log
autostart=true
autorestart=true
redirect_stderr=true
stopwaitsecs = 60
stopsignal=INT
{% endhighlight %}

Обновляем supervisor

{% highlight bash %}
sudo supervisorctl update
{% endhighlight %}

Пеперь мы можем проверить статус приложения командой

{% highlight bash %}
sudo supervisorctl status
{% endhighlight %}

и перезапустить

{% highlight bash %}
sudo supervisorctl restart project
{% endhighlight %}

Теперь осталось настроить nginx для работы со всеми этими чудесами Сейчас нужно создать конфигурационный файл для nginx

{% highlight bash %}
sudo nano /etc/nginx/sites-available/project.conf
{% endhighlight %}

его содержимое

{% highlight nginx %}
server {
        listen       80;
        server_name site.com;
        access_log  /var/log/nginx/access.log  main;
        error_log   /var/log/nginx/error.log info;


       location / {
                uwsgi_pass 127.0.0.1:49001;
                include uwsgi_params;
       }

        location /media/ {
                alias /home/project/app/media/;
                expires 30d;
        }

        location /static/ {
                alias /home/project/app/static/;
                expires 30d;
        }
}
{% endhighlight %}

и нужно его активировать

{% highlight bash %}
ln -s /etc/nginx/sites-available/project.conf  /etc/nginx/sites-enabled/project.conf
{% endhighlight %}

Теперь перезапускаем nginx и все должно работать

{% highlight bash %}
sudo service nginx restart
{% endhighlight %}
