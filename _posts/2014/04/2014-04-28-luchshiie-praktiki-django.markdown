---
layout: post
title: Лучшие практики Django
date: '2014-04-28 12:29:00'
tags:
- django
---

Недавно наткнулся на статью о лучших практиках разработки на django (<a href="http://lincolnloop.com/django-best-practices/projects.html">ссылка</a>), меня сильно порадовало что большинство из этого я интуитивно использую уже довольно давно, но вот некоторые части мне кажется нужно делать подругому нежели описанны в статье. Сейчас хочу привести вашему вниманию симбеоз статьи указанной выше и моих мыслей. Также советую совместить данную статью с моим постом о [структуре django проекта]({{ site.baseurl }}{% post_url 2013/09/2013-09-01-django_structure %})
<!--more-->

### Струкура настроек проекта

Я обычно храню все основные настройки в файле <strong>settings.py</strong> и в него импортирую файл <strong>local_settings.py</strong>, также файл settings.py хранится в git, а local_settings.py делается для каждого окружения свой, но в git хранится local_settings.py.example который содержит структуру файла и то что можно в нем менять или добавить.

Импорт в settings.py:

{% highlight python %}
try:
    from settings_local import *
except ImportError:
    pass
{% endhighlight %}

Примерное содержимое файла local_settings.py.example:

{% highlight python %}
import os

DEBUG = True
TEMPLATE_DEBUG = DEBUG

ADMINS = (
    ('You', 'your@email'),
)
MANAGERS = ADMINS

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': os.path.join(os.path.realpath(__file__), 'dev.db'),
    }
}
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.dummy.DummyCache',
    }
}
{% endhighlight %}

Структура выглядит примерно так:

{% highlight bash %}
project #директория проекта
- prjct #директория основного приложения
- - settings.py
- - local_settings.py.example
- - local_settings.py
- - …
- manage.py
{% endhighlight %}

При переносе приложения на другой сервер, либо при открытии проекта другим разарботкичом требуется лишь переименовать local_settings.py.example в local_settings.py и внести нужные настройки.

### Использование путей к файлам.

В настройках django нужно указывать пути до директорий в которых хранятся статические файлы, наблоны, файлы локализации и тд. Для этого не нужно писать полный путь, а лучше поручить все это python. Я для этого использую данный прием:

{% highlight python %}
import os

PROJECT_DIR = os.path.dirname(os.path.realpath(__file__))

def p(*args):
    return os.path.join(PROJECT_DIR, '..', *args)
{% endhighlight %}

В теперь путь функция p будет указывать на корень проекта, и использовать ее нужно вот так:

{% highlight python %}
STATIC_ROOT = p('static’)
{% endhighlight %}

Для статических файлов будет определена директория static в корне проекта.

### Статические файлы в шаблоне.

Для определения путей к статическим файлам я использую приложение staticfiles (django.contrib.staticfiles), его нужно добавить к списку INSTALLED_APPS:

{% highlight python %}
INSTALLED_APPS = [
….
 ‘django.contrib.staticfiles’,
….
]
{% endhighlight %}

И теперь во всех шаблонах мы сможем использовать удобную конструкцию для загрузки статических файлов:

{% highlight python %}
{% raw %}
{% load static %}
<img src="{% static «images/test.jpg" %}" />
{% endraw %}
{% endhighlight %}

### Работа с шаблонами.

Самый лучший и однозначный способ именования шаблонов это использование такой конструкции:

{% highlight python %}
{приложение} / {модель} _ {функция}. HTML
{% endhighlight %}

Функции:
* <strong>list</strong> - для вывода списков
* <strong>detail</strong> - для детальное отображения
* <strong>form</strong> - для вывода формы

А все части шаблона нужно складывать в директорию includes

Я указал лишь основы, в будущем скорей всего продолжу тему хороших практик при создании django  приложения
