---
layout: post
title: SFTP сервер с помощью Django
date: '2019-02-11 15:22:38'
tags:
- django
- sftp
---

Не спрашивайте зачем, но я занялся созданием [рабочего SFTP сервера для Django](https://github.com/vahaah/django-sftp). Он позволяет работать с django-storages и использовать S3.

На данный момент это альфа версия, вернее даже proof of concepts. Но в ближайшее время все докручу

<!--more-->

Сейчас все очень просто, ставится из PyPi:

{% highlight bash %}
pip install django-sftp
{% endhighlight %}

Добавляем в `INSTALLED_APPS`

{% highlight python %}
INSTALLED_APPS = (
        ...
        'django_sftp',
        ...
    )

{% endhighlight %}

Проводим миграции:

{% highlight bash %}
./manage.py migrate django_sftp
{% endhighlight %}

Генерируем ключ:

{% highlight bash %}
ssh-keygen -t rsa -b 4096 -C "your_email@example.com" -m PEM
{% endhighlight %}

И запускаем на нужном порте:

{% highlight bash %}
./manage.py sftpserver :11121 -k /tmp/rsa
{% endhighlight %}