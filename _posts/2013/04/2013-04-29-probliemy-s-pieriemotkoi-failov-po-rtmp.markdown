---
layout: post
title: Проблемы с перемоткой файлов по RTMP
date: '2013-04-29 12:06:00'
tags:
- dev
- nginx
---

Иногда бывает так, что после сохранения видео файла не работает перемотка при использовании rtmp сервера вещания, сейчас я расскажу как с этим справится
<!--more-->
Проблема чаще всего заключается в том что нарушается мета информация видео файла и нам необходимо ее востановить. Для этого прекрасно подходит пакет Yamdi - Yet Another MetaData Injector for FLV.

Для начала нам нужно поставить сам Yamdi, для этого перейдем в папку <strong>build</strong> которую мы создали когда ставили nginx-rtmp и начинается копипаст (:

{% highlight bash %}
wget http://citylan.dl.sourceforge.net/project/yamdi/yamdi/1.9/yamdi-1.9.tar.gz
tar xzvf yamdi-1.9.tar.gz
cd yamdi-1.9
make
make install
{% endhighlight %}

теперь мы можем использовать пакет через <strong>/usr/local/bin/yamdi</strong>

Теперь мы можем востановить мета информацию файла набрав следующую команду:

{% highlight bash %}
/usr/local/bin/yamdi -i video.flv -o video_with_meta.flv -s -k
{% endhighlight %}

И вот основные команды:

* M - Удаление мета-данных
* s - Add the onLastSecond event
* k - Add the onLastKeyframe event
* w - переписать существующий файл
* c - добавить комментарий
