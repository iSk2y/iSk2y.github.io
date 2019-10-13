---
title: 从SandBox(03)谈类视图的装饰
date: 2018-11-10 22:59:20
tags:
  - Django
  - Python
categories: Django
---



从[SandBox（03）](https://zhuanlan.zhihu.com/p/48419456)谈类视图的装饰

<!-- more -->

# 前言



## SandBox谈到

页面访问限制的实现需求：

- 用户登录系统才可以访问某些页面 - 如果用户没有登陆而直接访问就会跳转到登陆界面 - 用户在跳转的登陆页面完成登陆后，自动访问跳转前的访问地址

新建sandboxMP/apps/system/mixin.py，写入如下内容：

```python
from django.contrib.auth.decorators import login_required


class LoginRequiredMixin(object):
    @classmethod
    def as_view(cls, **init_kwargs):
        view = super(LoginRequiredMixin, cls).as_view(**init_kwargs)
        return login_required(view)
```

修改sandboxMP/sandboxMP/settings.py, 加入LOGIN_URL

```python
LOGIN_URL = '/login/'
```

需要登入用户才能访问的视图，只需要继承LoginRequiredMixin即可，修改后的IndexView视图如下：

```python
from .mixin import LoginRequiredMixin

class IndexView(LoginRequiredMixin, View): 

    def get(self, request):
        return render(request, 'index.html')
```

注意：LoginRequiredMixin位于继承列表最左侧位置



## 自我思考

从上面可以看到，其实是新建了一个LoginRequiredMixin的类，然后让IndexView来继承这个类。在这个LoginRequiredMixin类中也写了一个`as_view()`方法。这相当于重载了IndexView的`as_view()`的方法。这样也起到了装饰类的视图的效果。



# 类视图

趁着这个机会来对照[官方文档2.1](https://docs.djangoproject.com/zh-hans/2.1/topics/class-based-views/intro/) [中文1.11文档 ](https://yiyibooks.cn/xx/Django_1.11.6/topics/class-based-views/intro.html)看看类视图的装饰方法。



## 起源

>## 通用视图，基于类的视图和基于类的通用视图的关系和历史[¶](https://yiyibooks.cn/__trs__/xx/Django_1.11.6/topics/class-based-views/intro.html#the-relationship-and-history-of-generic-views-class-based-views-and-class-based-generic-views)
>
>开始的时候只有视图函数，Django 传递一个[`HttpRequest`](https://yiyibooks.cn/__trs__/xx/Django_1.11.6/ref/request-response.html#django.http.HttpRequest) 给你的函数并期待返回一个[`HttpResponse`](https://yiyibooks.cn/__trs__/xx/Django_1.11.6/ref/request-response.html#django.http.HttpResponse)。 Django 曾经提供的就这么些内容。
>
>在早期，我们认识到在视图开发过程中有共同的用法和模式。 这时我们引入基于函数的通用视图来抽象这些模式以简化常见情形的视图开发。
>
>基于函数的视图的问题在于，虽然它们很好地覆盖了简单的情形，但是不能扩展或自定义它们，即使是一些简单的配置选项，这让它们在现实应用中受到很多限制。
>
>基于类的通用视图然后应运而生，目的与基于函数的通用视图一样，就是为了使得视图的开发更加容易。 然而，它们使用的Mixin解决办法使得基于类的通用视图比基于函数的视图更加容易扩展和更加灵活。
>
>如果你以前使用基于函数的通用视图并发现它们的不足，你不能认为基于类的通用视图只是简单地用基于类的方法实现一个等价的替代，你应该认为它们是解决原始问题的一个全新的方法。
>
>Django 使用基类和Mixin来构建基于类的通用视图,为用户提供了最大的灵活性，默认的方法包含很多属性和方法的钩子，但是简单的用法中不需要考虑他们，也可以正常工作。 例如，不是将您限制为`form_class`的基于类的属性，而是使用调用`get_form_class`方法的`get_form`方法，其默认实现只返回类的`form_class`属性。 这给你多种选择来指定具体使用的表单，例如一个属性或者一个完全动态的、可调用的钩子。 这些选择似乎白白地增加了简单使用场景的复杂性，但是没有它们更高级的功能就会受到限制。

## 优势

- 组织与特定HTTP方法相关的代码（`GET`，`POST`等） 可以通过单独的方法而不是条件分支来解决。
- 面向对象的技术例如Mixin（多继承）可以将代码分解成可重用的组件。



## 示例

基于类的视图的核心是允许你用不同的实例方法来响应不同的HTTP 请求方法，而不是在一个视图函数中使用条件分支代码来实现。例子如下：

```python
from django.http import HttpResponse
from django.views import View

class MyView(View):
    def get(self, request):
        # <view logic>
        return HttpResponse('result')
```

因为Django的URL解析器希望将请求和关联的参数发送到可调用函数，而不是类，基于类的视图具有一个[`as_view()`](https://yiyibooks.cn/__trs__/xx/Django_1.11.6/ref/class-based-views/base.html#django.views.generic.base.View.as_view)类方法，它返回一个可以在请求时调用的函数到达与相关模式匹配的URL。 该函数创建一个类的实例并调用其[`dispatch()`](https://yiyibooks.cn/__trs__/xx/Django_1.11.6/ref/class-based-views/base.html#django.views.generic.base.View.dispatch)方法。 `dispatch` 查看请求是`GET` 还是`POST` 等等，并将请求转发给相应的方法，如果该方法没有定义则引发[`HttpResponseNotAllowed`](https://yiyibooks.cn/__trs__/xx/Django_1.11.6/ref/request-response.html#django.http.HttpResponseNotAllowed)：

```python
# urls.py
from django.conf.urls import url
from myapp.views import MyView

urlpatterns = [
    url(r'^about/$', MyView.as_view()),
]
```



## as_view方法

*classmethod* `as_view`(**\*initkwargs*)[¶](https://docs.djangoproject.com/zh-hans/2.1/ref/class-based-views/base/#django.views.generic.base.View.as_view)

返回一个可调用的视图，它接受一个请求并返回一个响应：

```
response = MyView.as_view()(request)
```

The returned view has `view_class` and `view_initkwargs` attributes.

When the view is called during the request/response cycle, the [`HttpRequest`](https://docs.djangoproject.com/zh-hans/2.1/ref/request-response/#django.http.HttpRequest) is assigned to the view's `request` attribute. Any positional and/or keyword arguments [captured from the URL pattern](https://docs.djangoproject.com/zh-hans/2.1/topics/http/urls/#how-django-processes-a-request) are assigned to the `args` and `kwargs` attributes, respectively. Then [`dispatch()`](https://docs.djangoproject.com/zh-hans/2.1/ref/class-based-views/base/#django.views.generic.base.View.dispatch) is called.

下面我们来看下源代码

```python
@classonlymethod
    def as_view(cls, **initkwargs):
        """Main entry point for a request-response process."""
        for key in initkwargs:
            if key in cls.http_method_names:
                raise TypeError("You tried to pass in the %s method name as a "
                                "keyword argument to %s(). Don't do that."
                                % (key, cls.__name__))
            if not hasattr(cls, key):
                raise TypeError("%s() received an invalid keyword %r. as_view "
                                "only accepts arguments that are already "
                                "attributes of the class." % (cls.__name__, key))

        def view(request, *args, **kwargs):
            self = cls(**initkwargs)
            if hasattr(self, 'get') and not hasattr(self, 'head'):
                self.head = self.get
            self.request = request
            self.args = args
            self.kwargs = kwargs
            return self.dispatch(request, *args, **kwargs)
        view.view_class = cls
        view.view_initkwargs = initkwargs

        # take name and docstring from class
        update_wrapper(view, cls, updated=())

        # and possible attributes set by decorators
        # like csrf_exempt from dispatch
        update_wrapper(view, cls.dispatch, assigned=())
        return view
```

他其实是返回了view的句柄，而定义的view就是dispatch，并把request和参数都一并传入了，那dispatch是干什么的？



## dispatch方法

`dispatch`(*request*, **args*, **\*kwargs*)[¶](https://docs.djangoproject.com/zh-hans/2.1/ref/class-based-views/base/#django.views.generic.base.View.dispatch)

The `view` part of the view -- the method that accepts a `request` argument plus arguments, and returns a HTTP response.

The default implementation will inspect the HTTP method and attempt to delegate to a method that matches the HTTP method; a `GET` will be delegated to `get()`, a `POST` to `post()`, and so on.

By default, a `HEAD` request will be delegated to `get()`. If you need to handle `HEAD` requests in a different way than `GET`, you can override the `head()` method. See [Supporting other HTTP methods](https://docs.djangoproject.com/zh-hans/2.1/topics/class-based-views/#supporting-other-http-methods) for an example.

从官方文档给出的解释就是检查HTTP方法并把对应方式调度给对应方法，来看看源码

```python
    http_method_names = ['get', 'post', 'put', 'patch', 'delete', 'head', 'options', 'trace']
    def dispatch(self, request, *args, **kwargs):
        # Try to dispatch to the right method; if a method doesn't exist,
        # defer to the error handler. Also defer to the error handler if the
        # request method isn't on the approved list.
        if request.method.lower() in self.http_method_names:
            handler = getattr(self, request.method.lower(), self.http_method_not_allowed)
        else:
            handler = self.http_method_not_allowed
        return handler(request, *args, **kwargs)
```

判断method是否存在与允许的http_method_names中，如果允许利用反射的方法也就是getattr去取到self的函数句柄，然后return回来执行结果。

这样就很清晰了原来是这样执行的过程，那我们接着来看如何装饰它



# 装饰类的视图

## 装饰as_view()

> 装饰基于类的视图的最简单的方法是装饰[`as_view()`](https://yiyibooks.cn/__trs__/xx/Django_1.11.6/ref/class-based-views/base.html#django.views.generic.base.View.as_view) 方法的结果。 最方便的地方是URLconf 中部署视图的位置：

```python
from django.contrib.auth.decorators import login_required, permission_required
from django.views.generic import TemplateView

from .views import VoteView

urlpatterns = [
    url(r'^about/$', login_required(TemplateView.as_view(template_name="secret.html"))),
    url(r'^vote/$', permission_required('polls.can_vote')(VoteView.as_view())),
]
```

文档中说到直接在urlconf中用login_required来装饰as_view()就可以了。其实sandbox(03)中创建了类再重载as_view()目的和效果 与这个一致的，同样是装饰了as_view()返回的view，其实也就是dispatch。



这个方法可以在具体实例基础上运用了装饰器，如果要让某个视图的每个实例都被装饰那就看下面的装饰类。



## 装饰类

若要装饰基于类的视图的每个实例，需要装饰类本身。 可以将装饰器运用到类的[`dispatch()`](https://yiyibooks.cn/__trs__/xx/Django_1.11.6/ref/class-based-views/base.html#django.views.generic.base.View.dispatch) 方法上来实现这点。

类的方法和独立的函数不完全相同，所以不可以直接将函数装饰器运用到方法上 —— 首先需要将它转换成一个方法装饰器。 `method_decorator` 装饰器将函数装饰器转换成方法装饰器，这样它就可以用于实例方法上。 像这样：

```python
from django.contrib.auth.decorators import login_required
from django.utils.decorators import method_decorator
from django.views.generic import TemplateView

class ProtectedView(TemplateView):
    template_name = 'secret.html'

    @method_decorator(login_required)
    def dispatch(self, *args, **kwargs):
        return super(ProtectedView, self).dispatch(*args, **kwargs)
```

或者，更简洁的是，可以装饰类，并将要装饰的方法的名称作为关键字参数`name`传递：

```python
@method_decorator(login_required, name='dispatch')
class ProtectedView(TemplateView):
    template_name = 'secret.html'
```

在几个地方使用了一组常用的装饰器，您可以定义一个列表或元组的装饰器，并使用它，而不是多次调用`method_decorator()`。 这两个类是相当的：

```python
decorators = [never_cache, login_required]

@method_decorator(decorators, name='dispatch')
class ProtectedView(TemplateView):
    template_name = 'secret.html'

@method_decorator(never_cache, name='dispatch')
@method_decorator(login_required, name='dispatch')
class ProtectedView(TemplateView):
    template_name = 'secret.html'
```



# Reference

[基于类的视图简介](https://yiyibooks.cn/xx/Django_1.11.6/topics/class-based-views/intro.html)


