---
layout: post
title: Реализация простой корзины на Django
date: '2013-04-28 12:01:00'
tags:
- dev
- django
---

В последнии дни очень много людей спрашивали, как сделать корзину на Django сайте и вот я решил написать реализацию простейшей корзины для сайта.
<!--more-->
В первую очередь нам нужно описать модель нашей корзины

{% highlight python %}
#models.py
from django.db import models
from catalog.models import Product

class CartItem(models.Model):
    cart_id = models.CharField(max_length=50)
    date_added = models.DateTimeField(auto_now_add=True)
    quantity = models.IntegerField(default=1)
    product = models.ForeignKey(Product, unique=False)

    class Meta:
        ordering = ['date_added']

    def total(self):
        return self.quantity * self.product.price

    def name(self):
        return self.product.name

    def price(self):
        return self.product.price

    def augment_quantity(self, quantity):
         self.quantity = self.quantity + int(quantity)
         self.save()
{% endhighlight %}

Как мне кажется здесь все просто, корзина связана с нашим продуктом (Product) и все действия понятны и дополнительного описания не требуется.

Теперь давайте опишем логику работы корзины, для это в папке с приложением создадим файл файл cart.py

{% highlight python %}
#cart.py
import decimal
import random
from django.shortcuts import get_object_or_404
from cart.models import CartItem
from catalog.models import Product

CART_ID_SESSION_KEY = 'cart_id'


def _cart_id(request):
    if request.session.get(CART_ID_SESSION_KEY,'') == '':
        request.session[CART_ID_SESSION_KEY] = _generate_cart_id()
    return request.session[CART_ID_SESSION_KEY]


def _generate_cart_id():
    cart_id = ''
    characters = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz1234567890!@#$%^&amp;*() '
    cart_id_length = 50
    for y in range(cart_id_length):
        cart_id += characters[random.randint(0, len(characters)-1)]
    return cart_id


def get_cart_items(request):
    return CartItem.objects.filter(cart_id=_cart_id(request))


def add_to_cart(request):
    postdata = request.POST.copy()
    product_slug = postdata.get('product_slug', '')
    quantity = postdata.get('quantity', 1)
    p = get_object_or_404(Product, slug=product_slug)
    cart_products = get_cart_items(request)
    product_in_cart = False
    for cart_item in cart_products:
        if cart_item.product.id == p.id:
            cart_item.augment_quantity(quantity)
            product_in_cart = True
    if not product_in_cart:
        ci = CartItem()
        ci.product = p
        ci.quantity = quantity
        ci.cart_id = _cart_id(request)
        ci.save()


def cart_distinct_item_count(request):
    return get_cart_items(request).count()


def get_single_item(request, item_id):
    return get_object_or_404(CartItem, id=item_id, cart_id=_cart_id(request))


def update_cart(request):
    postdata = request.POST.copy()
    item_id = postdata['item_id']
    quantity = postdata['quantity']
    cart_item = get_single_item(request, item_id)
    if cart_item:
        if int(quantity) &gt; 0:
            cart_item.quantity = int(quantity)
            cart_item.save()
        else:
            remove_from_cart(request)


def remove_from_cart(request):
    postdata = request.POST.copy()
    item_id = postdata['item_id']
    cart_item = get_single_item(request, item_id)
    if cart_item:
        cart_item.delete()


def cart_subtotal(request):
    cart_total = decimal.Decimal('0.00')
    cart_products = get_cart_items(request)
    for cart_item in cart_products:
        cart_total += cart_item.product.price * cart_item.quantity
    return cart_total


def is_empty(request):
    return cart_distinct_item_count(request) == 0


def empty_cart(request):
    user_cart = get_cart_items(request)
    user_cart.delete()

{% endhighlight %}

Здесь тоже все понятно и все описанно в названии функций, теперь сделаем форму для корзины

{% highlight python %}
#forms.py
from django import forms
class ProductAddToCartForm(forms.Form):
	quantity = forms.IntegerField(widget=forms.TextInput(attrs={'size':'2',
		'value':'1', 'class':'input-small', 'maxlength':'5'}),
		error_messages={'invalid':'Введите правильное количество'}, min_value=1, label='Количество')
	product_slug = forms.CharField(widget=forms.HiddenInput())
	def __init__(self, request=None, *args, **kwargs):
		self.request = request
		super(ProductAddToCartForm, self).__init__(*args, **kwargs)
	def clean(self):
		if self.request:
			if not self.request.session.test_cookie_worked():
				raise forms.ValidationError("Cookies должны быть включены")
		return self.cleaned_data
{% endhighlight %}

И представление

{% highlight python %}
from django.shortcuts import render_to_response
from django.template import RequestContext
from django.http import HttpResponseRedirect

from cart import cart

def show_cart(request, template_name="cart/cart.html"):
    if request.method == 'POST':
        postdata = request.POST.copy()
        if postdata['submit'] == 'Remove':
            cart.remove_from_cart(request)
        if postdata['submit'] == 'Update':
            cart.update_cart(request)
    cart_items = cart.get_cart_items(request)
    page_title = 'Корзина'
    cart_subtotal = cart.cart_subtotal(request)
    return render_to_response(template_name, locals(), context_instance=RequestContext(request))
{% endhighlight %}

Ну и urls.py для полной картины

{% highlight python %}
    urlpatterns = patterns('cart.views',
	url(r'^$', 'show_cart', { 'template_name': 'cart/cart.html'}, 'show_cart'),
	)
{% endhighlight %}

Думаю шаблон описывать не стоит, так как все делается также как и для любых других представлений.

Даже если вы просто скопипастите данные скрипты у вас будет полностью рабочая корзины и вчитавшись в код вы сможете ее модернизировать под свои нужды
