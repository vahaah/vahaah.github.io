---
layout: post
title: Использование docker-gen и nginx-proxy для своего docker хостинга
date: '2016-07-19 09:22:38'
tags:
- nginx
- docker
---

Для докера есть потрясающий пакет под названием [docker-gen](https://github.com/jwilder/docker-gen) он позволяет получать мета информацию о запущенных контейнерах, сигналы старта/завершения контейнера и еще много приятных мелочей. В этой статье я хочу описать создание простого paas на основе docker-gen. Наш сервис сможет создавать конфиг nginx для всех запущенных проектов в конечном результате у нас будет свой маленький heroku и мы загружать несколько разных проектов на один сервер с помощью docker-machine.

<!--more-->

Для начала создадим docker-compose.yml для нашего nginx:

{% highlight yaml %}
version: '2'
services:
  nginx-proxy:
    image: jwilder/nginx-proxy
    container_name: nginx-proxy
    ports:
      - "80:80"
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock:ro
{% endhighlight %}

Я использую готовый пакет [nginx-proxy](https://github.com/jwilder/nginx-proxy) для генерации конфигурационного файла nginx в общем можно обойтись и без него, но для первой статьи так будет проще.

Далее нам нужно создать network через docker, чтобы наш docker-gen мог просматривать все контейнеры и у nginx был к ним доступ:

{% highlight bash %}
docker network create nginx-proxy
{% endhighlight %}
Теперь объявим эту сеть в нашем docker-compose файле:

{% highlight yaml %}
version: '2'
services:
  nginx-proxy:
    image: jwilder/nginx-proxy
    container_name: nginx-proxy
    ports:
      - "80:80"
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock:ro

networks:
  default:
    external:
      name: nginx-proxy
{% endhighlight %}

Теперь нужно создать docker-machine и выполнить `docker-compose up`.

Основа нашего PaaS дана давайте создадим docker-compose.yml для нашего первого проекта:

{% highlight yaml %}
version: '2'
services:
  host:
    image: jwilder/whoami
    environment:
      - VIRTUAL_HOST=host1.local

networks:
  default:
    external:
      name: nginx-proxy

{% endhighlight %}

Здесь я использовал образ [whoami](https://github.com/jwilder/whoami) который возвращает id контейнера и добавил переменную окружения VIRTUAL_HOST которая содержит домен для данного проекта. Ну и подключился к ранее созданной сети.

Теперь нужно подключиться к нашей docker-machine и выполнить `docker-compose up`. И если мы перейдем на host1.local то увидим сообщение **I'm d54d3d39423b** где d54d3d39423b id вашего контейнера.

Теперь мы можем создать еще один проект с VIRTUAL_HOST=host2.local:

{% highlight yaml %}
version: '2'
services:
  host:
    image: jwilder/whoami
    environment:
      - VIRTUAL_HOST=host2.local

networks:
  default:
    external:
      name: nginx-proxy

{% endhighlight %}

И также его запустить и мы увидим 2 одновременно запущенный проекта на одном хосте. Можно также выполнить команду для масштабирования нашего проекта `docker-compose scale host=2` и если мы начнем обновлять страницу host2.local то увидим что id будет изменяться.

Теперь у нас есть свой мини хостинг на который мы можем загружать простые статический сайты через docker-machine.

На этом я закончу. В следующей части я опишу более приближенный к реальности пример.
