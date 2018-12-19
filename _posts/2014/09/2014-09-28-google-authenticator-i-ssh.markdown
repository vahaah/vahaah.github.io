---
layout: post
title: Google Authenticator и SSH
date: '2014-09-28 12:31:00'
tags:
- ubuntu
---

В последнее время у меня повышенная параноя по поводу моих серверов и я решил как то дополнительно себя обезопасить, и средством безопасности я выбрал Google Authenticator, подумалось мне что одноразовые пароли затруднят все задачи по взлому “моих сокровищ” (:

<!--more-->

### Установка Google Authenticator

Все действия я проводил на <strong>Ubuntu Server 12.04 x64</strong>. Этот пакет есть в репозиториях Ubuntu поэтому установка проходит как обычно без всяких проблем:

{% highlight bash %}
sudo apt-get install libpam-google-authenticator
{% endhighlight %}

### Создание авторизационного ключа

После того как мы все установили остаемся под основным пользователем и набираем в консоли команду

{% highlight bash %}
google-authenticator
{% endhighlight %}

После этого у вас появится ссылка на QR код и собственно сам QR

Для того, чтобы добавить серверв свое приложение на сматртфоне нужно считать QR код либо ввести <strong>secret key</strong> и все будет работать, дальше нажимаем <strong>“y”</strong> там еще будет серия вопросов, на них можно также ответь <strong>“y”</strong> или прочитать и решить самому нужно оно или нет.


### Активация Google Authenticator

Теперь нужно добавить Google Authenticator как способ авторизации по SSH. Для этого нужно отредактировать файл /etc/pam.d/sshd

{% highlight bash %}
sudo nano /etc/pam.d/sshd
{% endhighlight %}

И добавить строку:

{% highlight bash %}
auth required pam_google_authenticator.so
{% endhighlight %}

Далее редактируем файл <strong>/etc/ssh/sshd_config</strong> и меняем настройку <strong>ChallengeResponseAuthentication</strong> на <strong>yes</strong>

{% highlight bash %}
ChallengeResponseAuthentication yes
{% endhighlight %}

Теперь перезапускаем SSH

{% highlight bash %}
sudo service ssh restart
{% endhighlight %}

И вуаля, у нас теперь при каждой авторизации по SSH требуется код Google Authenticator
