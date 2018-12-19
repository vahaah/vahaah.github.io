---
layout: post
title: О settings.py и хранении "секретных" настроек
date: '2015-02-28 13:32:00'
tags:
- dev
- django
---

Я уже писал о настройках в этом [посте]({{ site.baseurl }}{% post_url 2014/04/2014-04-28-luchshiie-praktiki-django %}). Но в последнее время я пересмотрел свое мнение по поводу хранения настроек, мне кажется использование файла local_settings.py избыточно и ведет к лишнему копипасту при работе с проектом. В последнее время я перел к такому формату:

<!--more-->

{% highlight python %}
settings/
    __init__.py
     base.py
     local.py
     dev.py
     production.py
     test.py
     ….
{% endhighlight %}

Работает это так, все основные настройки хранятся в base.py и мы используем нужные настройки для нужного окружения, примеру local.py будет выглядеть вот так:

{% highlight python %}
# -*- coding: utf-8 -*-
from .base import *

BASE_HOST = "http://example.com/"
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': ‘project',
        'USER': 'root',
        'PASSWORD': '',  
        'HOST': 'localhost',
        'PORT': ''
    }
}

CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.dummy.DummyCache',
    }
}


DEBUG = True
TEMPLATE_DEBUG = DEBUG
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'

INSTALLED_APPS += ('debug_toolbar')</code>
{% endhighlight %}

И для запуска тестового сервера нам нужно ввести команду:

{% highlight python %}
python manage.py runserver --settings=project.settings.local
{% endhighlight %}

Теперь все конфигурационный файлы для всех окружений можно хранить в системе контроля версий, ненужно создавать лишних файлов в окружении.

Теперь можно плавно перейти к хранению секретных настроек, таких как SECRET_KEY, логин/пароль от бд, логин/пароль от почтового сервера и тд. Как мы все понимаем хранение это информации в контроле версий крайне не правильна, поэтому мне кажется лучше это хранить в системный переменных, для этого нужно задать нужную информацию в .bashrc, .bash_profile или .profile или если вы используете virtualenvs то достаточно дописать нужную информацию в bin/activate, к примеру мы хотим задать SECRET_KEY и пароль от БД:

{% highlight bash %}
export MY_SECRET_KEY=$)a7n&o80u!6y5t-+j
export MY_BD_PASS=qweasd
{% endhighlight %}

Как же получить доступ к этим настройка из django? Да все просто все это есть в модуле os:

{% highlight python %}
import os

SECRET_KEY = os.environ&#91;‘MY_SECRET_KEY'&#93;
{% endhighlight %}

Но это не очень красиво и нужно отлавливать ошибки, поэтому правильней написать функцию:

{% highlight python %}
import os
from django.core.exceptions import ImpropelyConfigured


def get_env_setting(var):
     try:
          return os.environ&#91;var&#93;
     excapt KeyError:
          msg = u”Переменная %s не задана” % var
          raise ImpropelyConfigured(msg)

SECRET_KEY = get_env_setting(‘MY_SECRET_KEY’)</code>
{% endhighlight %}

Теперь все красиво и безопасно.
