---
layout: post
title: Драйвер docker-machine для vscale
date: '2016-04-23 13:42:52'
tags:
- docker
- vscale
- go
---

Начал пользоваться сервисом [Vscale](http://vscale.io), но неудобно каждый раз создавть инстанс и работать с ним через generic-driver. Конечно же начал поиск уже готового драйвера и наткнулся на [docker-machine-vscale](https://github.com/evrone/docker-machine-vscale), но к сожалению он никак не хотел собираться и issue об этом висело уже 1,5 месяца, решил его "отремотировать" и вот результат [docker-machine-driver-vscale](https://github.com/vahaah/docker-machine-driver-vscale). Вначале хотел отправить просто pull request, но по причине спешки вместо форка сделал свой репозиторий и дальше стало просто лень все переделывать.

<!--more-->

Теперь кратко об использовании модуля.

Установка:

{% highlight bash %}
$ curl -L https://github.com/vahaah/docker-machine-driver-vscale/releases/download/1.0.0/docker-machine-driver-vscale > /usr/local/bin/docker-machine-driver-vscale
$ chmod +x /usr/local/bin/docker-machine-driver-vscale
{% endhighlight %}

Последняя версия находится [здесь](https://github.com/vahaah/docker-machine-driver-vscale/releases)

Дальше нужно сгенерировать Token для работы с Vscale, это можно сделать [в профиле](https://vscale.io/panel/settings/tokens/).

Ну и использование:
{% highlight bash %}
$ docker-machine create -d vscale --vscale-access-token YOUR_VSCALE_ACCESS_TOKEN machine_name
{% endhighlight %}

Для простоты работы вы можете использовать переменые окружения:

{% highlight bash %}
$ export VSCALE_LOCATION=YOUR_VSCALE_ACCESS_TOKEN
$ docker-machine create -d vscale machine_name
{% endhighlight %}

Досупные опции:
<table><thead>
<tr>
<th style="min-width: 180px;">Опция</th>
<th>Переменная окружения</th>
<th>Значение по умолчанию</th>
<th>Доступные значения</th>
</tr>
</thead><tbody>
<tr>
<td><code>--vscale-access-token</code></td>
<td><code>VSCALE_ACCESS_TOKEN</code></td>
<td>-</td>
<td>-</td>
</tr>
<tr>
<td><code>--vscale-location</code></td>
<td><code>VSCALE_LOCATION</code></td>
<td><code>spb0</code></td>
<td>spb0</td>
</tr>
<tr>
<td><code>--vscale-rplan</code></td>
<td><code>VSCALE_RPLAN</code></td>
<td><code>small</code></td>
<td>
   small<br>
   medium <br>
   large <br>
   huge <br>
   monster
</td>
</tr>
<tr>
<td><code>--vscale-made-from</code></td>
<td><code>VSCALE_MADE_FROM</code></td>
<td><code>ubuntu_14.04_64_002_master</code></td>
<td>debian_8.1_64_001_master<br>
    centos_7.1_64_001_master<br>
    ubuntu_14.04_64_002_master
</td>
</tr>
</tbody></table>
