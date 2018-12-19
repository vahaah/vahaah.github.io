---
layout: post
title: Установка GitLab и Redmine на Ubuntu 12.04
date: '2013-06-10 12:16:00'
tags:
- ubuntu
- dev
---

Для начала скажу пару слов о том что есть что. GitLab, на мой взгляд, это лучшая веб обертка над git, и если быть совсем кратким это свой личный GitHub. Redmine это отличный менеджер проектов. Как мне кажется это две очень необходимые вещи для любого разработчика
<!--more-->
Для начала необходимо установить все нужные пакеты:

{% highlight bash %}
apt-get update
apt-get upgrade
apt-get install -y build-essential zlib1g-dev libyaml-dev libssl-dev \
libgdbm-dev libreadline-dev libncurses5-dev libffi-dev curl git-core \
openssh-server redis-server checkinstall libxml2-dev libxslt-dev \
libcurl4-openssl-dev libicu-dev postfix python nginx \
imagemagick  libmagickcore-dev libmagickwand-dev \
mysql-server mysql-client libmysqlclient-dev
{% endhighlight %}

Теперь нам нужно удалить старую версию ruby и установить то что нужно нам для работы

{% highlight bash %}
apt-get remove ruby1.8
{% endhighlight %}

и ставим новый ruby

{% highlight bash %}
mkdir /tmp/ruby && cd /tmp/ruby
curl --progress http://ftp.ruby-lang.org/pub/ruby/1.9/ruby-1.9.3-p429.tar.gz | tar xz
cd ruby-1.9.3-p429
./configure
make
make install
gem install bundler --no-ri --no-rdoc
{% endhighlight %}

### Установка GitLab

для начала нужно создать нового пользователя для GitLab

{% highlight bash %}
adduser --disabled-login --gecos 'GitLab' git
{% endhighlight %}

меняем пользователя и переходим в домашнюю директорию:

{% highlight bash %}
sudo su git
cd /home/git
{% endhighlight %}

Устанавливаем gitlab-shell

{% highlight bash %}
git clone https://github.com/gitlabhq/gitlab-shell.git
cd gitlab-shell/
git checkout -b v1.5.0 v1.5.0
cp config.yml{.example,}
vim config.yml # проверяем конфигурацию
./bin/install
cd ../
{% endhighlight %}

Устанавливаем GitLab

{% highlight bash %}
cd /home/git
git clone https://github.com/gitlabhq/gitlabhq.git gitlab
cd /home/git/gitlab
git checkout -b v5.2.1 v5.2.1
cp config/gitlab.yml{.example,}
vim config/gitlab.yml # проверяем конфигурацию
chown -R git log/
chown -R git tmp/
mkdir tmp/pids/
mkdir tmp/sockets/
mkdir public/uploads
chmod -R u+rwX  log/
chmod -R u+rwX  tmp/
chmod -R u+rwX  public/uploads
cp config/puma.rb{.example,}
git config --global user.name "GitLab"
git config --global user.email "gitlab@example.com"
{% endhighlight %}

Создаем и настраиваем базу данных

{% highlight bash %}
cp config/database.yml{.mysql,}
mysql -u root -p

mysql> CREATE USER 'gitlab'@'localhost' IDENTIFIED BY '$password';
mysql> CREATE DATABASE IF NOT EXISTS &#96;gitlabhq_production&#96; DEFAULT CHARACTER SET &#96;utf8&#96; COLLATE &#96;utf8_unicode_ci&#96;;
mysql> GRANT SELECT, LOCK TABLES, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER ON &#96;gitlabhq_production&#96;.* TO 'gitlab'@'localhost';
mysql> exit;

vim config/database.yml
{% endhighlight %}

И теперь проводим саму установку

{% highlight bash %}
gem install charlock_holmes --version '0.6.9.4' --no-ri --no-rdoc
bundle install --deployment --without development test postgres
bundle exec rake gitlab:setup RAILS_ENV=production
bundle exec rake gitlab:env:info RAILS_ENV=production
bundle exec rake sidekiq:start RAILS_ENV=production
{% endhighlight %}

Добавляем запуск службы в init.d

{% highlight bash %}
su
cp lib/support/init.d/gitlab /etc/init.d/gitlab
chmod +x /etc/init.d/gitlab
update-rc.d gitlab defaults 21
{% endhighlight %}

Поздравляю, мы только что установили GitLab

### Redmine

Для начала необходимо создать директорию и скачать само приложение

{% highlight bash %}
mkdir -m 755 /home/project
cd /home/project
git clone https://github.com/redmine/redmine.git
cd redmine
{% endhighlight %}

Конфигурируем базу данных:

{% highlight bash %}
cp config/database.yml{.example,}
mysql -u root -p

mysql> CREATE USER 'redmine'@'localhost' IDENTIFIED BY '$password';
mysql> CREATE DATABASE IF NOT EXISTS &#96;redmine&#96; DEFAULT CHARACTER SET &#96;utf8&#96; COLLATE &#96;utf8_unicode_ci&#96;;
mysql> GRANT SELECT, LOCK TABLES, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER ON &#96;redmine&#96;.* TO 'redmine'@'localhost';
mysql> exit;

vim config/database.yml # вносим информацию о бд
{% endhighlight %}

Для запуска redmine нам необходимо скопировать файл puma.rb из GitLab

{% highlight bash %}
cp /home/git/gitlab/config/puma.rb config/puma.rb
vim config/puma.rb # изменить все gitlab на redmine, и откоректировать пути
vim Gemfile # нужно добавить строку " gem "puma", '~> 2.0.1' "
cp -pR config/configuration.yml{.example,}
vim config/configuration.yml # основная конфигурация

bundle install --without development test postgresql sqlite
rake generate_secret_token
{% endhighlight %}

И последние шаги:

{% highlight bash %}
RAILS_ENV=production rake db:migrate
RAILS_ENV=production rake redmine:load_default_data
mkdir tmp public/plugin_assets
chown -R www-data:www-data /home/project
chmod -R 755 files log tmp public/plugin_assets
{% endhighlight %}

Теперь нам нужно создать файл /etc/init.d/redmine и добавить в него следующее содержимое <a href="https://gist.github.com/vahaah/6146231">Gist</a>

{% highlight bash %}
vim /etc/init.d/redmine
chmod +x /etc/init.d/redmine
update-rc.d redmine defaults 21
{% endhighlight %}

### NGINX

Конфигурация nginx для gitlab

{% highlight nginx %}
vim /etc/nginx/sites-available/gitlab


upstream gitlab {
  server unix:/home/git/gitlab/tmp/sockets/gitlab.socket;
}

server {
  listen 80;         
  server_name git.smidth.ru;
  server_tokens off;
  root /home/git/gitlab/public;

  access_log  /var/log/nginx/gitlab_access.log;
  error_log   /var/log/nginx/gitlab_error.log;

  location / {
    try_files $uri $uri/index.html $uri.html @gitlab;
  }

  location @gitlab {
    proxy_read_timeout 300;
    proxy_connect_timeout 300;
    proxy_redirect     off;

    proxy_set_header   X-Forwarded-Proto $scheme;
    proxy_set_header   Host              $http_host;
    proxy_set_header   X-Real-IP         $remote_addr;

    proxy_pass http://gitlab;
  }
}
{% endhighlight %}

и конфигурация для redmine

{% highlight nginx %}
vim /etc/nginx/sites-available/redmine


upstream redmine {
  server unix:/home/project/redmine/tmp/sockets/redmine.socket;
}

server {
  listen 80;         
  server_name redmine.smidth.ru;
  server_tokens off;
  root /home/project/redmine/public;

  access_log  /var/log/nginx/redmine_access.log;
  error_log   /var/log/nginx/redmine_error.log;

  location / {
    try_files $uri $uri/index.html $uri.html @redmine;
  }

  location @redmine {
    proxy_read_timeout 300;
    proxy_connect_timeout 300;
    proxy_redirect     off;

    proxy_set_header   X-Forwarded-Proto $scheme;
    proxy_set_header   Host              $http_host;
    proxy_set_header   X-Real-IP         $remote_addr;

    proxy_pass http://redmine;
  }
}
{% endhighlight %}

После этого все что нам остается это перезапустить все службы

{% highlight bash %}
service gitlab restart
service redmine restart
service nginx restart
{% endhighlight %}
