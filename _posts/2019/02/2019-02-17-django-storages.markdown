---
layout: post
title: Использование django storages в своем приложении
date: '2019-02-17 15:00:38'
tags:
- django
- django-sftp
---

Мне нужно было обеспечить поддежку, совместимость и быстрое добавление новых типов файлового хранилища для [django-sftp](https://github.com/vahaah/django-sftp). Опять же решение взял у [django-ftpserver](https://github.com/tokibito/django-ftpserver)

<!--more-->

Начнем из далека у django есть встроеная функция для определения типа файлогово хранилища которое указано в setting.py

{% highlight python %}
>>> from django.core.files.storage import get_storage_class
>>> get_storage_class()
<class django.core.files.storage.FileSystemStorage>
{% endhighlight %}

С именем хранилища которое мы используем в системе мы можем написать небольшой класс который может сделать патч основного класса нашей фаловой системы.

{% highlight python %}
class StoragePatch:
    patch_methods = []
    
    @classmethod
    def apply(cls, fs):
        """replace bound methods of fs.
        """
        fs._patch = cls
        for method_name in cls.patch_methods:
            # if fs hasn't method, raise AttributeError.
            origin = getattr(fs, method_name)
            method = getattr(cls, method_name)
            bound_method = method.__get__(fs, fs.__class__)
            setattr(fs, method_name, bound_method)
            setattr(fs, '_origin_' + method_name, origin)
{% endhighlight %}

Остновная идея в том чтобы не перегружать все методы, а только те которые указаны в `patch_methods` и сохранить исходный метод с пометкой `'_origin_' + method_name`.


Пример для Django's FileSystemStorage

{% highlight python %}
class FileSystemStoragePatch(StoragePatch):
    """StoragePatch for Django's FileSystemStorage.
    """
    patch_methods = (
        'mkdir', 'rmdir',
    )
    
    def mkdir(self, path):
        os.mkdir(self.storage.path(path))
    
    def rmdir(self, path):
        os.rmdir(self.storage.path(path))
{% endhighlight %}

И соответственно в нашем главном классе мы можем сделать вот так:

{% highlight python %}
class StorageFS(object):

    storage = get_storage_class()

    patches = {
        'FileSystemStorage': FileSystemStoragePatch,
        'S3Boto3Storage': S3Boto3StoragePatch
    }
    
    def apply_patch(self):
        patch = self.patches.get(self.storage.__class__.__name__)
        if patch:
            patch.apply(self)
{% endhighlight %}

В общем все просто и работает :)