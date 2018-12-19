---
layout: post
title: Создание собственной модели User в Django
date: '2013-05-14 12:11:00'
tags:
- dev
- django
---

​С приходом <strong>django 1.5</strong> у нас наконецто появилась делать собственные модели <strong>User</strong>, сейчас я продемонстрирую вам как сделать простейшую модель.
<!--more-->
Для начала давайте создадим наше приложение:

{% highlight bash %}
python ../manage.py startapp accounts
{% endhighlight %}

Теперь мы может отредактировать модель для нашего пользователя, открывает файл <strong>project.accounts.models</strong> для начала нам нужно добавить импорт нужных нам модулей, в данном случае хватит

{% highlight python %}
from django.db import models
from django.contrib.auth.models import BaseUserManager,AbstractBaseUser
{% endhighlight %}

Теперь мы можем делать собствено модель пользователя

{% highlight python %}
class User(AbstractBaseUser):
    email = models.EmailField(
        verbose_name='Электропочта',
        max_length=255,
        unique=True,
        db_index=True,)
    username = models.CharField(verbose_name='Ник',  max_length=255, unique=True)
    avatar = models.ImageField(verbose_name='Аватар',  upload_to='images/%Y/%m/%d', blank=True, null=True)
    first_name = models.CharField(verbose_name='Имя',  max_length=255, blank=True)
    last_name = models.CharField(verbose_name='Фамилия',  max_length=255, blank=True)
    date_of_birth = models.DateField(verbose_name='День рождения',  blank=True, null=True)
    is_active = models.BooleanField(default=True)
    is_admin = models.BooleanField(default=False)
{% endhighlight %}

Здесь нет ничего сложного все как обычно, теперь немного магии и выберем поле для входа на сайт и поля которые мы будем проверять, для этого добавим следующие строки

{% highlight python %}
    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = ['username']
{% endhighlight %}

Теперь давайте создадим модель для администраторов, для этого над классом <strong>User</strong> создадим класс <strong>UserManager</strong>

{% highlight python %}
class UserManager(BaseUserManager):
    def create_user(self, email, username, password=None):
        if not email:
            raise ValueError('Users must have an email address')

        user = self.model(
            email=UserManager.normalize_email(email),
            username=username,
            )

        user.set_password(password)
        user.save(using=self._db)
        return user

    def create_superuser(self, email, username, password):
        user = self.create_user(email,
                                password=password,
                                username=username
        )
        user.is_admin = True
        user.save(using=self._db)
        return user
{% endhighlight %}

И также в класс <strong>User</strong> нужно добавить следующую строку

{% highlight python %}
    objects = UserManager()
{% endhighlight %}

Теперь давайте создадим функции в классе <strong>User</strong> которые облегчат нам жизнь

{% highlight python %}
    def get_full_name(self):
        return '%s %s' % (self.first_name, self.last_name,)

    def get_short_name(self):
        return self.username

    def __unicode__(self):
        return self.email

    def has_perm(self, perm, obj=None):
        return True

    def has_module_perms(self, app_label):
        return True

    @property
    def is_staff(self):
        return self.is_admin
{% endhighlight %}

Собственно из названий функций сразу видно зачем они нужны

Полный вариант <strong>project.accounts.models</strong>

{% highlight python %}
# -*- coding: utf-8 -*-
from django.db import models
from django.contrib.auth.models import BaseUserManager, AbstractBaseUser


class UserManager(BaseUserManager):
    def create_user(self, email, username, password=None):
        if not email:
            raise ValueError('Users must have an email address')

        user = self.model(
            email=UserManager.normalize_email(email),
            username=username)

        user.set_password(password)
        user.save(using=self._db)
        return user

    def create_superuser(self, email, username, password):
        user = self.create_user(email,
                                password=password,
                                username=username)
        user.is_admin = True
        user.save(using=self._db)
        return user


class User(AbstractBaseUser):
    email = models.EmailField(
        verbose_name='Электропочта',
        max_length=255,
        unique=True,
        db_index=True)
    username = models.CharField(verbose_name='Ник',  max_length=255, unique=True)
    avatar = models.ImageField(verbose_name='Аватар',  upload_to='images/%Y/%m/%d', blank=True, null=True)
    first_name = models.CharField(verbose_name='Имя',  max_length=255, blank=True)
    last_name = models.CharField(verbose_name='Фамилия',  max_length=255, blank=True)
    date_of_birth = models.DateField(verbose_name='День рождения',  blank=True, null=True)
    is_active = models.BooleanField(default=True)
    is_admin = models.BooleanField(default=False)

    objects = UserManager()

    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = ['username']

    def get_full_name(self):
        return '%s %s' % (self.first_name, self.last_name,)

    def get_short_name(self):
        return self.username

    def __unicode__(self):
        return self.email

    def has_perm(self, perm, obj=None):
        return True

    def has_module_perms(self, app_label):
        return True

    @property
    def is_staff(self):
        return self.is_admin
{% endhighlight %}

Давайте сразу создадим необходимые формы т.е редактируем файл <strong>project.account.forms</strong>

{% highlight python %}
# -*- coding: utf-8 -*-
from django import forms
from project.accounts.models import User
from django.contrib.auth.forms import ReadOnlyPasswordHashField


class UserCreationForm(forms.ModelForm):
    password1 = forms.CharField(label='Password', widget=forms.PasswordInput)
    password2 = forms.CharField(label='Password confirmation', widget=forms.PasswordInput)

    class Meta:
        model = User
        fields = ('email', 'username')

    def clean_password2(self):
        password1 = self.cleaned_data.get("password1")
        password2 = self.cleaned_data.get("password2")
        if password1 and password2 and password1 != password2:
            raise forms.ValidationError("Passwords don't match")
        return password2

    def save(self, commit=True):
        user = super(UserCreationForm, self).save(commit=False)
        user.set_password(self.cleaned_data["password1"])
        if commit:
            user.save()
        return user


class UserChangeForm(forms.ModelForm):
    password = ReadOnlyPasswordHashField()

    class Meta:
        model = User

    def clean_password(self):
        return self.initial["password"]
{% endhighlight %}

Теперь приступим к редактированию<strong> project.accounts.admin</strong> и наполним его следующим содержимым

{% highlight python %}
# -*- coding: utf-8 -*-
from django.contrib import admin
from django.contrib.auth.models import Group
from django.contrib.auth.admin import UserAdmin

from project.accounts.models import User
from project.accounts.forms import UserChangeForm, UserCreationForm


class UserAdmin(UserAdmin):
    form = UserChangeForm
    add_form = UserCreationForm


    list_display = ('email', 'username', 'is_admin',)
    list_filter = ('is_admin',)
    fieldsets = (
        (None, {'fields': ('email', 'username', 'password')}),
        ('Personal info', {'fields': ('date_of_birth', 'first_name', 'last_name', 'avatar')}),
        ('Permissions', {'fields': ('is_admin',)}),
        ('Important dates', {'fields': ('last_login',)}),
    )
    add_fieldsets = (
        (None, {
            'classes': ('wide',),
            'fields': ('email', 'date_of_birth', 'password1', 'password2')}
        ),
    )
    search_fields = ('email',)
    ordering = ('email',)
    filter_horizontal = ()

admin.site.register(User, UserAdmin)
admin.site.unregister(Group)
{% endhighlight %}

Теперь осталась маленькая деталька, это добавление нашей модели в <strong>settings.py</strong>

{% highlight python %}
AUTH_USER_MODEL = 'accounts.User'
{% endhighlight %}

Вот теперь поздравляю Вас с созданием собственной модели пользователя сайта
