---
layout: post
title: Letsencrypt и nginx на Ubuntu 14.04
date: '2016-02-29 13:16:03'
tags:
- ubuntu
- nginx
---

Хочу рассказать как настроить Letsencrypt на Ubuntu 14.04 совместно с nginx и сделать автопродление сертификатов.

<!--more-->

Начнем с того что нужно поставить git

{% highlight bash %}
apt-get -y install git
{% endhighlight %}

Далее скачиваем сам letsencrypt:

{% highlight bash %}
git clone https://github.com/letsencrypt/letsencrypt /opt/letsencrypt
{% endhighlight %}

Теперь нужно проверить что 80 порт доступен для этого нужно отключить nginx

{% highlight bash %}
service nginx stop
{% endhighlight %}

Ну и убедимся что все свободно и ничего нам не мешает:

{% highlight bash %}
service nginx stop
{% endhighlight %}

Теперь преступим к выпуску сертификата, мы будем использовать Standalone плагин для этого:

{% highlight bash %}
cd /opt/letsencrypt
./letsencrypt-auto certonly --standalone
{% endhighlight %}

Нам нужно будем ввести домен к которому мы выпустим сертификат (если нужно выпустить сертификаты к нескольким доменам для разделения нужно использовать "пробел")

После этого действия у нас появится наш первый сертификат, по умолчанию все сертификаты хранятся в директории `/etc/letsencrypt/live/имя_домен`.

Теперь заставим все это работать с nginx. Для начала настроим переадресацию на https
{% highlight nginx %}
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}
{% endhighlight %}

И конфигурация для 443 порта:

{% highlight nginx %}
server {
    listen 443;
    server_name example.com www.example.com;

    ssl on;
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_ciphers 'EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH';
}
{% endhighlight %}
Теперь все работает, но как вы можете видеть срок жизни нашего сертификата всего месяц и очень не хочется каждый раз проделывать данные действия в ручную. Поэтому сейчас я опишу как настроить автопродление сертификатов.

Для этого мы будем использовать Webroot плагин. Он создает скрытую директорию `/.well-known` в корне вашего сайта используемую для валидации самим letsencrypt. Я предлагаю вынести эту директорию из корня всех сайтов, и расположить в `/var/www/`. Для этого нужно добавить в конф. файл nginx такой блок:

{% highlight nginx %}
location /.well-known/ {
    alias /var/www/.well-known/;
    allow all;
}
{% endhighlight %}

Теперь проверим все ли корректно работает:

{% highlight bash %}
cd /opt/letsencrypt
./letsencrypt-auto certonly -a webroot --agree-tos --renew-by-default --webroot-path=/var/www -d example.com -d www.example.com
{% endhighlight %}

И мы должны получить сообщение

{% highlight bash %}
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at
   /etc/letsencrypt/live/example.com/fullchain.pem. Your cert will expire
   on 2016-05-29. To obtain a new version of the certificate in the
   future, simply run Let's Encrypt again.
 - If you like Let's Encrypt, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
{% endhighlight %}

Теперь давайте приступим к автоматизации данного действия. Для начала нужно создать конфигурационый файл, мы скопируем его и примеров в самом letsencrypt

{% highlight bash %}
cp /opt/letsencrypt/examples/cli.ini /usr/local/etc/le-renew-webroot.ini
{% endhighlight %}

И отредактируем так как нам нужно

{% highlight bash %}
vi /usr/local/etc/le-renew-webroot.ini
{% endhighlight %}

На выходе мы должны получить такой конфиг:

{% highlight ini %}
rsa-key-size = 4096

email = you@example.com

domains = example.com, www.example.com

webroot-path = /var/www
{% endhighlight  %}

Теперь проверим наш конфигурационный файл:

{% highlight bash %}
cd /opt/letsencrypt
./letsencrypt-auto certonly -a webroot --renew-by-default --config /usr/local/etc/le-renew-webroot.ini
{% endhighlight bash %}
Теперь создадим скрипт для автоматического выполнения продления если нашему сертификату осталось жить меньше 30 дней. К своему счастью я нашел готовое решение [gist.github.com/thisismitch](https://gist.github.com/thisismitch/e1b603165523df66d5cc).

Здесь делаем все просто

{% highlight bash %}
curl -L -o /usr/local/sbin/le-renew-webroot https://gist.githubusercontent.com/thisismitch/e1b603165523df66d5cc/raw/fbffbf358e96110d5566f13677d9bd5f4f65794c/le-renew-webroot
chmod +x /usr/local/sbin/le-renew-webroot
{% endhighlight %}

Проверяем наш скрипт:

{% highlight bash %}
le-renew-webroot
{% endhighlight %}

И добавляем его в crontab

{% highlight bash %}
crontab -e
{% endhighlight %}

{% highlight bash %}
30 2 * * 1 /usr/local/sbin/le-renew-webroot >> /var/log/le-renewal.log
{% endhighlight %}
Лог всех действий будет доступен в `/var/log/le-renewal.log`

Вот собственно и все. Также в процессе написания нашел на просторах более полную и подробно расписанную инструкцию на [английском ](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-14-04) собственно часть со скриптом обновления и утащил из нее
