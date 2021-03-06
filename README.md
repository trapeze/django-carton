
Django Carton
=============


      +------+
     /|     /|
    +-+----+ |    django-carton is a simple and lightweight application
    | |    | |    for shopping carts and wish lists.
    | +----+-+
    |/     |/
    +------+



* Simple: You decide how to implement the views, templates and payment
  processing.
* Lightweight: The cart lives in the session.
* Just a container: You define your product model the way you want.


Usage Example
-------------

View:

    from django.http import HttpResponse

    from carton.cart import Cart
    from products.models import Product

    def add(request):
        cart = Cart(request.session)
        product = Product.objects.get(id=request.GET.get('product_id'))
        cart.add(product, price=product.price)
        return HttpResponse("Added")

    def show(request):
        return render(request, 'shopping/show-cart.html')


We are assuming here that your products are defined in an application
called ``products``.

Template:

    {% load carton_tags %}
    {% get_cart as cart %}

    {% for item in cart.items %}
        {{ item.product.name }}
        {{ item.quantity }}
        {{ item.total_price }}
    {% endfor %}


This project is shipped with an application example called ``shopping``
implementing basic add, remove, display features.
To use it, you will need to install the ``shopping`` application and
include the URLs in your project ``urls.py``

    # settings.py
    INSTALLED_APPS = (
        'carton',
        'shopping',
        'products',
    )

    # urls.py
    urlpatterns = patterns('',
        url(r'^shopping-cart/', include('shopping.urls')),
    )


Assuming you have some products defined, you should be able to
add, show and remove products like this:

    /shopping-cart/add/?id=1
    /shopping-cart/show/
    /shopping-cart/remove/?id=1


Installation
------------

Just install the package using something like pip and add ``carton`` to
your ``INSTALLED_APPS`` setting.

This is how you run tests:

    ./manage.py test tests --settings=carton.tests.settings


Abstract
--------

The cart is an object that's stored in session. Products are associated
to cart items.

    Cart
    |-- CartItem
    |----- product
    |----- price
    |----- quantity

A cart item stores a price, a quantity and an arbitrary instance of
a product model.


You can access all your product's attributes, for instance it's name:

    {% for item in cart.items %}
        {{ item.price }}
        {{ item.quantity }}
        {{ item.product.name }}
    {% endfor %}



Managing Cart Items
-------------------

These are simple operations to add, remove and access cart items:

    >>> apple = Product.objects.all()[0]
    >>> cart.add(apple, price=1.5)
    >>> apple in cart
    True
    >>> cart.remove(apple)
    >>> apple in cart
    False

    >>> orange = Product.objects.all()[1]
    >>> cart.add(apple, price=1.5)
    >>> cart.total
    Decimal('1.5')
    >>> cart.add(orange, price=2.0)
    >>> cart.total
    Decimal('3.5')


Note how we check weather the product is in the cart - The following
statements are different ways to do the same thing:

    >>> apple in cart
    >>> apple in cart.products
    >>> apple in [item.product for item in cart.items]


The "product" refers to the database object. The "cart item" is where
we store a copy of the product, it's quantity and it's price.

    >>> cart.items
    [CartItem Object (apple), CartItem Object (orange)]

    >>> cart.products
    [<Product: apple>, <Product: orange>]


Clear all items:

    >>> cart.clear()
    >>> cart.total
    0


Increase the quantity by adding more products:

    >>> cart.add(apple, price=1.5)
    >>> cart.add(apple)  # no need to repeat the price.
    >>> cart.total
    Decimal('3.0')

Note that the price is only needed when you add a product for the first time.

    >>> cart.add(orange)
    *** ValueError: Missing price when adding a cart item.


You can add several products at the same time:

    >>> cart.clear()
    >>> cart.add(orange, price=2.0, quantity=3)
    >>> cart.total
    Decimal('6')
    >>> cart.add(orange, quantity=2)
    >>> cart.total
    Decimal('10')


The price is relevant only the first time you add a product:

    >>> cart.clear()
    >>> cart.add(orange, price=2.0)
    >>> cart.total
    Decimal('2')
    >>> cart.add(orange, price=100)  # this price is ignored
    >>> cart.total
    Decimal('4')


Note how the price is ignored on the second call.


You can change the quantity of product that are already in the cart:

    >>> cart.add(orange, price=2.0)
    >>> cart.total
    Decimal('2')
    >>> cart.set_quantity(orange, quantity=3)
    >>> cart.total
    Decimal('6')
    >>> cart.set_quantity(orange, quantity=1)
    >>> cart.total
    Decimal('2')
    >>> cart.set_quantity(orange, quantity=0)
    >>> cart.total
    0
    >>> cart.set_quantity(orange, quantity=-1)
    *** ValueError: Quantity must be positive when updating cart



Removing all occurrence of a product:

    >>> cart.add(apple, price=1.5, quantity=4)
    >>> cart.total
    Decimal('6.0')
    >>> cart.remove(apple)
    >>> cart.total
    0
    >>> apple in cart
    False


Remove a single occurrence of a product:

    >>> cart.add(apple, price=1.5, quantity=4)
    >>> cart.remove_single(apple)
    >>> apple in cart
    True
    >>> cart.total
    Decimal('4.5')
    >>> cart.remove_single(apple)
    >>> cart.total
    Decimal('3.0')
    >>> cart.remove_single(apple)
    >>> cart.total
    Decimal('1.5')
    >>> cart.remove_single(apple)
    >>> cart.total
    0


Settings
--------

### Template Tag Name

You can retrieve the cart in templates using
`{% get_cart as my_cart %}`.

You can change the name of this template tag using the
`CART_TEMPLATE_TAG_NAME` setting.


    # In you project settings
    CART_TEMPLATE_TAG_NAME = 'get_basket'

    # In templates
    {% load carton_tags %}
    {% get_basket as my_basket %}


### Stale Items

Cart items are associated to products in the database.
Products that were added to the cart and then removed from the
database are called stale items. The `CART_REMOVE_STALE_ITEMS` defines
whether stales items should be automatically removed from the cart. By
default they are removed.

### Session Key

The `CART_SESSION_KEY` settings controls the name of the session key.
