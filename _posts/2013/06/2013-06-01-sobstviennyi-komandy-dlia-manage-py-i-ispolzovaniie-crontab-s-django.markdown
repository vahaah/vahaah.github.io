---
layout: post
title: Собственный команды для manage.py и использование crontab с Django
date: '2013-06-01 12:14:00'
tags:
- dev
- django
---

Для одного проекта необходимо было сделать автоматическое создание материала раз в сутки и собственно сам контент нужно было брать со стороннего api. В этом посте я расскажу как создать свои команды для manage.py которые можно использовать в crontab
<!--more-->
Для примера я буду использовать обновления курса доллара с ЦБ РФ. Сначала опишем нашу модель, она будет максимально простой, будет только значение курса и дата

{% highlight python %}
class Curs(models.Model):
	value = models.FloatField(u"Значение")
	date = models.DateTimeField(u"Дата создания", auto_now_add=True)
{% endhighlight %}

Теперь нам нужно привести структуру нашего приложения к такому виду:

{% highlight bash %}
valute/
    __init__.py
    models.py
    management/
        __init__.py
        commands/
            __init__.py
            updatecurs.py
    tests.py
    views.py
{% endhighlight %}

Теперь давайте напишем какие действия будут проходить по команде. Собственно сам файл updatecurs.py

{% highlight python %}
from django.core.management.base import BaseCommand, CommandError
from valute.models import Curs
from grab import Grab


class Command(BaseCommand):
    help = 'Update curs'

    def handle(self, *args, **options):
		g = Grab()
		g.go(''http://www.cbr.ru/scripts/XML_daily.asp')
		value = g.xpath_one(u'//valute[@id="R01010"]/value/text()')
		curs = Curs(value=value)
		curs.save()
		self.stdout.write('New curs "%s"' % value
{% endhighlight %}

Здесь как мне кажется все понятно, мы используем библиотеку Grab для того, чтобы взять значение курса и создаем новую запись в базе и по окончании в консоле выводим значение курса. Теперь для использования этого действия мы можем использовать команду updatecurs в manage.py.

И теперь мы можем добавить запись в наш crontab, чтобы обновлять курс валюты каждый день

{% highlight bash %}
5 0 */1 * * python /path/to/manage.py updatecurs
{% endhighlight %}
