---
layout: post
title: Динамическое создание сериализаторов в Django Rest Framework
date: '2016-06-01 17:51:23'
tags:
- django
- python
- drf
- rest
---

Для одной задачи мне пришлось создавать сериализаторы DRF на лету. Все достаточно просто и особо не отличается от создания django форм или моделей.

<!--more-->

Если совсем кратко то все делается с помощью встроенной функции type и выглядит вот так:

{% highlight python %}
serializer = type(name, (serializers.ModelSerializer,), attrs)
{% endhighlight %}

Теперь подробней. Допустим у нас есть модель:

{% highlight python %}
class Person(models.Model):
    first_name = models.CharField(max_length=255)
    last_name = models.CharField(max_length=255)
{% endhighlight %}

И из нее нам нужно сделать сериализатор, для этого объявим в функцию в которую мы будем передавать 2 аргумента это сама модель и нужные нам поля:

{% highlight python %}
def dynamic_serializer(model, fields=None):
    name = "{0}Serializer".format(model.__name__)

    class Meta:
        pass

    setattr(Meta, 'model', model)
    if fields:
        setattr(Meta, 'fields', fields)

    attrs = {'Meta': Meta}
    return type(name, (serializers.ModelSerializer,), attrs)
{% endhighlight %}

Как мы знаем модель в serializers.ModelSerializer указывается в классе Meta, поэтому мы его просто объявляем и указываем нужные атрибуты с помощью setattr, самый важное указать модель которую мы будем использовать, но также мы сделали возможность использовать только нужные поля модели в сериализаторе

Пример использованиям

{% highlight python %}
serializer = dynamic_serializer(Person) # Для вывода всех полей модели
serializer = dynamic_serializer(Person, ['first_name']) # Только для нужных нам полей
{% endhighlight %}

Также в словарь attrs можно добавить нужные нам свойства класса, в общем любое свойство которое нужно объявить в сериализаторе.
