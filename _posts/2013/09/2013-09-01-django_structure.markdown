---
layout: post
title: Структура Django проекта
date: '2013-09-01 12:24:00'
tags:
- dev
- django
---

Все мы знаем что есть очень много вариантов структуры для <strong>Django</strong> проекта и большинство из них вполне нормальные, но я хочу рассказать как я храню проекты, и вообще структуру приложений в проекте

<!--more-->

Для примера наш проект будет называться <strong>prjct</strong>. Для названий других примеров я буду добавлять к названию число, т.е. <strong>prjct1</strong>, <strong>prjct2</strong> и тд. Для названия приложений я буду такую же структуру <strong>app0</strong>, <strong>app1</strong>, <strong>app2</strong> и тд. В качестве пользователя будем использовать мифического юзера django

### Структура проектов в директории ~/
Для хранения проектов я использую домашнюю директорию, никаких /var, /srv. Все проекты хранятся в директории <code>~/projects/</code>:

{% highlight bash %}
~/projects/
        project/
        project1/
        project2/
        …..
{% endhighlight %}

Каждый проект имеет свое виртуальное окружение, виртуальное окружение называется также как проект, все виртуальные окружения хранятся в директории <code>~/.envs/</code> :

{% highlight bash %}
~/.envs/
        project/
        project1/
        project2/
        …..
{% endhighlight %}

т.е. для доступа к любому виртуальному окружению проекта нужно ввести `source ~/.envs/<имя_проекта>/bin/activate` .

Все логи хранятся в директории <code>~/logs/</code>, как вы уже поняли логи каждого проекта хранятся в своей директории с именем проекта:

{% highlight bash %}
~/logs/
        project/
                errors.log
                ...
        project1/
                errors.log
                ...  
        project2/
                errors.log
                ...  
        …..
{% endhighlight %}

Всю статику и медиа файлы я храню в директории с проектом.

### Структура проекта

{% highlight bash %}
~/projects/
        project/
                apps/
                        app1/
                                admin.py
                                models.py
                                ….
                        app2/
                                admin.py
                                models.py
                                ….  
                        …
                conf/
                        nginx/
                                project.conf
                                ...
                        supervisor/
                                project.conf
                                ...
                project/
                        settings.py
                        urls.py
                        ...
                run/
                        run.sh
                        ...
                staticfiles/
                        css/
                                project.css
                                ...
                        js/
                                project.js
                                ...
                        …
                templates/
                        base.html
                        …    
                fabfile.py
                manage.py
                requirements.txt

                media/
                        …   
                static/
                        …  
{% endhighlight %}

Теперь объясняю по порядку:

<dl class="dl-horizontal">
        <dt>project/apps/</dt>
        <dd>здесь хранятся все приложения проекта, в самом приложении хранятся middleware, context_processors и в все остальные части нужные для этого приложения, все части нужные для проекта целиком хранятся в project/project/</dd>
        <dt>project/conf/</dt>
        <dd>здесь хранятся все конфигурационные файлы для nginx, supervisor и тд.</dd>       
        <dt>project/project/</dt>
        <dd>это приложение созданное при старте проекта, здесь хранятся settings, urls,  middleware, context_processors для всего проетка</dd>       
        <dt>project/run/</dt>
        <dd>скрипт запуска gunicorn</dd>     
        <dt>project/staticfiles/</dt>
        <dd>статика, которая будет собираться collectstatic (эта папка хранится в git)</dd>     
        <dt>project/templates/</dt>
        <dd>папка с шаблонами</dd>      
        <dt>fabfile.py</dt>
        <dd>файл для Fabric</dd>      
        <dt>requirements.txt</dt>
        <dd>файл с зависимостями</dd>     
        <dt>project/media/</dt>
        <dd>папка с медиа файлами (не хранится в git)</dd>        
        <dt>project/static/</dt>
        <dd>папка со всей статикой проекта после collectstatic (не хранится в git)</dd>
</dl>

В общем это все, благодаря такой структуре можно без проблем создавать скрипты для создания окружения, использовать один и тот же <strong>fabfile.py</strong> и развертывать окружение почти на автомате.
