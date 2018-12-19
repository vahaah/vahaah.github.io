---
layout: post
title: Настройка сервера для Django. Gunicorn + Nginx + Ubuntu
date: '2014-02-28 12:28:00'
tags:
- ubuntu
- django
- nginx
---


В этой заметке я хочу описать процесс настройки боевого сервера для хостинга Django проектов. И так нам потребуется 30 минут времени и сервер на  Ubuntu 12.*

<!--more-->

Для начала необходимо создать пользователя и запретить автоизацию из под root

{% highlight bash %}
adduser django
adduser django sudo
{% endhighlight %}

теперь необходимо зайти на сервер из под <strong>root'а</strong>

{% highlight bash %}
nano /etc/ssh/sshd_config
{% endhighlight %}

и выставляем `PermitRootLogin no`

Перезапускаем ssh

{% highlight bash %}
service ssh restart
{% endhighlight %}

Теперь нужно перезайти на сервер под пользователем <strong>django</strong> и с этого момента мы будем использовать команду <strong>sudo</strong>

Для удобной работой с файловой системой предлагаю поставить <strong>Midnight Commander</strong>

{% highlight bash %}
sudo apt-get install mc
sudo mc
{% endhighlight %}

Для удобной работы с репозиториями ставим <strong>python-software-properties</strong>

{% highlight bash %}
sudo apt-get install python-software-properties
{% endhighlight %}

Дабавляем репозитории Nginx

{% highlight bash %}
sudo add-apt-repository ppa:nginx/stable
sudo apt-get update
{% endhighlight %}

Ставим необходимые пакеты

{% highlight bash %}
sudo apt-get install nginx mysql-server mysql-client libmysqlclient-dev python-mysqldb python-pip python-virtualenv supervisor git-core redis-server gunicorn make g++ python-dev
{% endhighlight %}

Можно еще установить <strong>postfix</strong> для отправки email, но я чаще пользуюсь smtp

{% highlight bash %}
sudo apt-get install postfix
{% endhighlight %}

### Структура проектов

Для структуризации файлов в проекте мы будем пользоваться вот этой моей [заметкой]({{ site.baseurl }}{% post_url 2013/09/2013-09-01-django_structure %}).

А по поводу хранения самих проектов могу сказать следующее, в домашней папке создадим 3 директории

{% highlight bash %}
cd ~/
mkdir projects #папка с проектами
mkdir logs #папка с логами
mkdir .envs #папка с виртуальными окружениями
{% endhighlight %}

### Виртуальные окружения
Теперь давайте подготовим виртуальные окружения для старта, как я писал в заметке о структуре проектов мы будем называть окружение также как называется проект в нашем случае это <strong>prjct</strong>

{% highlight bash %}
cd ~/
virtualenv —prompt="prjct" .envs/prjct
{% endhighlight %}

И теперь мы можем использовать наше виртуальное окружение для проекта

{% highlight bash %}
source ~/.envs/prjct/bin/activate
{% endhighlight %}

### Создание БД в MySQL

Заходим в <strong>mysql под пользователем root</strong> и с паролем который мы указали когда ставили пакет

{% highlight mysql %}
mysql -u root -p PASSWORD
#создаем БД
prjctmysql> CREATE DATABASE IF NOT EXISTS &#96;prjct&#96; DEFAULT CHARACTER SET &#96;utf8&#96; COLLATE &#96;utf8_unicode_ci&#96;;
#создаем пользователя prjct
mysql> CREATE USER 'prjct'@'localhost' IDENTIFIED BY '$password’; # укажите пароль вместо $password
#Даем пользователю права на БД
mysql> GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER ON &#96;prjct&#96;.* TO 'prjct'@'localhost’;
mysql> exit;
{% endhighlight %}

Теперь вносим эти настройки в свой <strong>settings.py</strong> <em>(лучше конечно настройки БД вынести в отдельный файл settings_local.py, чтобы было проще при разработке)</em>

### Перенос проекта

Теперь копируем проект в директорию <strong>~/projects/prjct/</strong>. Вы можете перенести файлы использую sftp или же скопировать из своего репозитория, если вы используете git. После того как вы скопировали все файлы проекта, нужно выполнить установку всех зависимостей <em>(они должны быть записаны в файле requirements.txt)</em>

{% highlight bash %}
pip install -r requirements.txt
{% endhighlight %}

Дальше проверим все ли работает и выполним все служебные команды

{% highlight bash %}
python manage.py validate #если все хорошо продолжаем
python manage.py syncdb
python manage.py migrate
python manage.py collectstatic —noinput
{% endhighlight %}

Теперь нужно создать директорию в которой мы будем хранить все логи проекта:

{% highlight bash %}
mkdir ~/logs/prjct
{% endhighlight %}

### Настройка supervisor и gunicorn

Теперь установим настройки нашего проекта в <strong>supervisor</strong>. В директории с проетом у нас должна быть создана директория <strong>conf</strong> и в ней директория <strong>supervisor</strong> в которой лежит файл <strong>prjct.conf</strong>. Я всегда храню их в директории с проектом, чтобы их содержимое можно было контолировать и при переносе с сервера на сервер просто копировать.

Содержимое файла

{% highlight bash %}
[program:prjct]

command=sh /home/django/projects/prjct/run/run.sh
directory=/home/django/projects/prjct
user=django
autostart=true
autorestart=true
stderr_logfile=/home/django/logs/prjct/errors.log
stdout_logfile=/home/django/logs/prjct/access.log
{% endhighlight %}

Теперь отдаем файл настроек supervisor’у

{% highlight bash %}
cd ~/projects/prjct/conf/supervisor
sudo ln prjct.conf /etc/supervisor/conf.d/prjct.conf
sudo supervisorctl update
sudo supervisorctl status
{% endhighlight %}

После последней команды мы должны увидеть наш проект в списке и то что он не работает. А не работает он потому что у нас нет файла для запуска gunicorn <strong>/home/django/projects/prjct/run/run.sh</strong>, собственно его нужно создать и его содержимое будет таково:

{% highlight bash %}
#!/bin/sh

NAME=prjc
SRCDIR=prjc

GUNICORN=/usr/bin/gunicorn_django
HOMEDIR=/home/django

VIRTUALENV=${HOMEDIR}/.virtualenvs/${NAME}
PROJECTDIR=${HOMEDIR}/projects/${NAME}
SOCKFILE=/tmp/${NAME}.sock
MANAGE=${PROJECTDIR}/manage.py

. ${VIRTUALENV}/bin/activate
python ${MANAGE} run_gunicorn --bind unix:${SOCKFILE}
{% endhighlight %}

Теперь перезапускаем наше приложение и смотрим статус:

{% highlight bash %}
sudo supervisorctl restart prjct
sudo supervisorctl status
{% endhighlight %}
Поидее все должно работать (:

### Настройка NGINX

В директории conf у нас должна быть директория nginx где мы будем хранить конфигурационный файл для nginx. Собственно вот создержимое файла <strong>~/projects/prjct/conf/nginx/prjct.conf</strong>:

{% highlight nginx %}
upstream prjct_server{

server unix:/tmp/prjct.sock fail_timeout=0;
}

server {
    listen 80;
    server_name prjct_domain;
    if ($http_host = www.prjct_domain) {
        rewrite (.*) http://prjct_domain$1 permanent;
}


location / {
client_max_body_size 20m;

proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header Host $http_host;
proxy_redirect off;
proxy_pass http://prjct_server;
autoindex on;
}


location /media {
root /home/django/projects/prjct;
autoindex off;
expires max;
access_log off;
}

location /static {
root /home/django/projects/prjct;
autoindex off;
expires max;
access_log off;
}

access_log /var/log/nginx/prjct.access.log;
error_log /var/log/nginx/prjct.error.log;
{% endhighlight %}

Этот файл необходимо скопировать к конфигам nginx

{% highlight bash %}
cd ~/projects/prjct/conf/nginx
sudo ln prjct.conf /etc/nginx/sites-enabled/prjct.conf
{% endhighlight %}

И последнее что осталось это перезапустить nginx

{% highlight bash %}
sudo service nginx restart
{% endhighlight %}

И теперь все должно работать
