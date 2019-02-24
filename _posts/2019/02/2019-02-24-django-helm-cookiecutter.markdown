---
layout: post
title: Django Helm Cookiecutter
date: '2019-02-24 18:22:38'
tags:
- django
- cookiecutter
- k8s
- helm
---

По работе регулярно приходится создавать Helm чарты в том числе и для Django. Меня каждый раз напрягало копирование кода и удаление/создание отдельных конфигов для Celery, настройка бд или выбор существующей и все в таком духе.

Поэтому собрался и начал делать свой шаблон для cookiecutter который будет подерживать все эти настройки.
<!--more-->

В целом все как всегда для cookiecutter ставим/обновляем до последней версии:

{% highlight bash %}
pip install "cookiecutter>=1.4.0"
{% endhighlight %}

Запускае для нашего шаблона:

{% highlight bash %}
cookiecutter https://github.com/pydanny/django-helm-cookiecutter
{% endhighlight %}

Выбираем конфигурацию:

{% highlight bash %}
chart_name [My Awesome Chart]:
chart_slug [my_awesome_chart]:
chart_directory_name [chart]:
version [0.1.0]:
image_url [gitlab.example.com/group/project]:
project_url [https://example.com]:
DATABASE_URL [postges:///my_awesome_chart]:
{% endhighlight %}

Happy Helming