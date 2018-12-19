---
layout: post
title: Установка PIL на Ubuntu 12.04
date: '2013-05-16 12:13:00'
tags:
- ubuntu
- dev
---

Сегодня я расскажу как поставить PIL на ubuntu 12.04 32 или 64, так, чтобы все работало. Вообще это настоящая проблема что в большинстве случаев нельзя просто так взять и поставить PIL, всегда нужно производить дополнительные действия.
<!--more-->
Вообще это наверное самый часто встречающийся пост в блогах о Python и Django, но я напишу его и у себя, и так начнем...

Для начала поставим необходимые пакеты

{% highlight bash %}
sudo apt-get build-dep python-imaging
{% endhighlight %}

Теперь нам нужно сделать симлинки на билиотеки

{% highlight bash %}
sudo ln -s /usr/lib/`uname -i`-linux-gnu/libfreetype.so /usr/lib/
sudo ln -s /usr/lib/`uname -i`-linux-gnu/libjpeg.so /usr/lib/
sudo ln -s /usr/lib/`uname -i`-linux-gnu/libz.so /usr/lib/
{% endhighlight %}

Как вы поняли uname -i автоматически заменится типом архитектуры вашего компьютера

Ну и теперь мы можем поставить сам PIL

{% highlight bash %}
pip install PIL
{% endhighlight %}

И вы должны увидеть заветную запись в отчете о установке

{% highlight bash %}
--- TKINTER support available
--- JPEG support available
--- ZLIB (PNG/ZIP) support available
--- FREETYPE2 support available
{% endhighlight %}

Вот собственно и все, пользуйтесь на здоровье (:
