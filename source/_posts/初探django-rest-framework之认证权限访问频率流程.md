---
title: 初探django-rest-framework之认证权限访问频率流程
date: 2018-11-13 13:29:42
tags:
  - Python
  - Django
categories: Django
---

初探django-rest-framework之认证权限访问频率流程

接触Django也有一个多月了，认识和熟悉了基本流程后，开始接触django rest framework了，这里简要记录下这个流程。加油！！Fighting！！


<!-- more -->

# 前言

从[从SandBox(03)谈类视图的装饰](https://isk2y.github.io/2018/11/10/%E4%BB%8ESandBox-03-%E8%B0%88%E7%B1%BB%E8%A7%86%E5%9B%BE%E7%9A%84%E8%A3%85%E9%A5%B0/) 这篇博文中曾经提及过CBV 类视图的简要流程以及装饰方法，初次接触rest framework正好顺着这个方向来解析下认证（authenicate）、权限（permission）和访问频率（throttle）是怎么完成的？

## 环境

- Python：3.6.5

- OS：window10

- requirement：

  >Package             Version
  >------------------- -------
  >Django              2.1
  >djangorestframework 3.9.0
  >pip                 18.1
  >pytz                2018.7
  >setuptools          40.5.0
  >wheel               0.32.2

# APIview

在rest framework中写类视图 不再直接继承View而是继承自APIview了，那我们来看看这个APIview是怎么走的，先来看看APIview这个类的源码：

![APIview](https://upload-images.jianshu.io/upload_images/14657587-7c3e53b9a3126175.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这里就不一一列举出来的，源码自行查看吧。

可以看到APIview是直接继承自View视图的。所以猜测他里面给我们另外添加了某些验证。



## as_view()

在URLconf中使用类视图的时候

```python
urlpatterns = [
    path('admin/', admin.site.urls),
    path('index/', views.IndexView.as_view())
]
```

是执行as_view的方法，至于原因可以去看[从SandBox(03)谈类视图的装饰](https://isk2y.github.io/2018/11/10/%E4%BB%8ESandBox-03-%E8%B0%88%E7%B1%BB%E8%A7%86%E5%9B%BE%E7%9A%84%E8%A3%85%E9%A5%B0/) ，那我们来看看这个方法和View

```python
    def as_view(cls, **initkwargs):
        """
        Store the original class on the view function.

        This allows us to discover information about the view when we do URL
        reverse lookups.  Used for breadcrumb generation.
        """
        if isinstance(getattr(cls, 'queryset', None), models.query.QuerySet):
            def force_evaluation():
                raise RuntimeError(
                    'Do not evaluate the `.queryset` attribute directly, '
                    'as the result will be cached and reused between requests. '
                    'Use `.all()` or call `.get_queryset()` instead.'
                )
            cls.queryset._fetch_all = force_evaluation

        view = super(APIView, cls).as_view(**initkwargs)
        view.cls = cls
        view.initkwargs = initkwargs

        # Note: session based authentication is explicitly CSRF validated,
        # all other authentication is CSRF exempt.
        return csrf_exempt(view)
```

这里执行了super去调用了View的as_view，那么view其实返回的就是View中的view内容，也就是dispatch的句柄了

> View中的as_view()

```python
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





## dispatch()

既然返回的是dispatch的句柄，那我们就继续追入。这里很重要的一个点要搞清楚，每次调用方法时要从self自身及继承的类从头开始寻找，而不是寻找当前类！！这个要注意。

以dispatch为例，调用dispatch方法，从IndexView视图开始寻找，自身不存在则寻找APIview，APIview中存在则停止，即调用APIview中的dispatch方法，也就是说他其实对View的dispatch给override了。这也类的很重要的一个特点和优点。

> APIview中的dispatch

```python
    def dispatch(self, request, *args, **kwargs):
        """
        `.dispatch()` is pretty much the same as Django's regular dispatch,
        but with extra hooks for startup, finalize, and exception handling.
        """
        self.args = args
        self.kwargs = kwargs
        request = self.initialize_request(request, *args, **kwargs) 
        self.request = request 
        self.headers = self.default_response_headers  # deprecate?

        try:
            self.initial(request, *args, **kwargs)

            # Get the appropriate handler method
            if request.method.lower() in self.http_method_names:
                handler = getattr(self, request.method.lower(),
                                  self.http_method_not_allowed)
            else:
                handler = self.http_method_not_allowed

            response = handler(request, *args, **kwargs)

        except Exception as exc:
            response = self.handle_exception(exc)

        self.response = self.finalize_response(request, response, *args, **kwargs)
        return self.response
```



> View中的dispatch

```python
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

我们来看dispatch中的代码，和View中比较多了一些操作。

其中两个添加的重要操作是

```python
request = self.initialize_request(request, *args, **kwargs)
self.initial(request, *args, **kwargs)
```

那我们要去看看到底他干了什么。继续追踪

### initialize_request

```python
    def initialize_request(self, request, *args, **kwargs):
        """
        Returns the initial request object.
        """
        parser_context = self.get_parser_context(request)
        
        return Request(
            request,
            parsers=self.get_parsers(),
            authenticators=self.get_authenticators(),
            negotiator=self.get_content_negotiator(),
            parser_context=parser_context
        )
```

从函数的自述doc中可以了解到是做了些初始request对象操作

这里其实是返回了一个Request的对象，该对象封装了原有的request，还有重要的一点是authenticators（这个后面会讲到），我们先追入Request类的定义看一下

![Request类](https://upload-images.jianshu.io/upload_images/14657587-ea156f3d2ae19a1c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们在本节中关注的request的和authenticate分别的定义方式如图，

request原对象被赋值给了Request._request属性而authenticate是从get_authenticators()这个方法中获取到的，具体如何下面将认证时会讲到。

而这个返回的Request对象赋值给了调用dispatch的视图 self.request了（具体看上面dispatch源码）。至此其他暂不解释

### initial

我们在来看看这个方法的定义

```python
    def initial(self, request, *args, **kwargs):
        """
        Runs anything that needs to occur prior to calling the method handler.
        """
        self.format_kwarg = self.get_format_suffix(**kwargs)

        # Perform content negotiation and store the accepted info on the request
        neg = self.perform_content_negotiation(request)
        request.accepted_renderer, request.accepted_media_type = neg

        # Determine the API version, if versioning is in use.
        version, scheme = self.determine_version(request, *args, **kwargs) # 版本
        request.version, request.versioning_scheme = version, scheme 

        # Ensure that the incoming request is permitted
        self.perform_authentication(request) # 认证验证
        self.check_permissions(request) # 权限验证
        self.check_throttles(request) # 频率验证
```

这个方法就比较重要了，先简单注释中解释下，接下来开始对这几个作用解释



# 认证authenticate

正常执行as_view()，返回dispatch句柄后，访问URL执行dispatch方法，用反射的方法去执行对应类视图中的方法。

## perform_authentication

### 那到底什么时候认证的呢？

认证其实就是initial中的perform_authentication方法，我们追入来看一下：

```python
    def perform_authentication(self, request):
        """
        Perform authentication on the incoming request.

        Note that if you override this and simply 'pass', then authentication
        will instead be performed lazily, the first time either
        `request.user` or `request.auth` is accessed.
        """
        request.user
```

对没错，这个方法就只有一行代码 request.user。这里要提醒下 这个request已经不是原request了，不要忘了在initialize_request中已经返回Request对象，而dispatch中将Request赋值给了request，具体回看上面代码

那我们看看这个request.user到底执行了什么骚操作？追踪user如下：

```python
@property
    def user(self):
        """
        Returns the user associated with the current request, as authenticated
        by the authentication classes provided to the request.
        """
        if not hasattr(self, '_user'):
            with wrap_attributeerrors():
                self._authenticate()
        return self._user
```

发现这个user是property属性，然后用反射方法判断是否Request有_user，这里我们这个流程走下来并没有定义过，于是执行self.\_authenticate()方法 ，继续追入查看

```python
    def _authenticate(self):
        """
        Attempt to authenticate the request using each authentication instance
        in turn.
        """
        for authenticator in self.authenticators: 
            # authenicators已经在initialize_request方法中给self定义过了
            try:
                user_auth_tuple = authenticator.authenticate(self) # 执行认证类中的方法 认证
            except exceptions.APIException:
                self._not_authenticated() # 报错就执行这个方法
                raise

            if user_auth_tuple is not None: # 如果上面认证方法返回的不是none
                self._authenticator = authenticator # 把认证的类给赋值给self
                self.user, self.auth = user_auth_tuple # 然后把认证返回后的元组 一个给user 一个给auth
                return

        self._not_authenticated()
```

我给上面的代码加了注释，更好理解。



发现这个方法是遍历了self.authenticators，try了authenticator.authenticate(self)这个方法，那这个self.authenticators时候定义的呢？对，注释里我写了在initialize_request这个方法中定义Request是赋值了，那我们回去看看到底是赋值了什么

```python
return Request(
            request,
            parsers=self.get_parsers(),
            authenticators=self.get_authenticators(),
            negotiator=self.get_content_negotiator(),
            parser_context=parser_context
        )
```

从上面的initialize_request中讲到在定义Request对象的时候传入的authenticate，可以看到，他是调用了get_authenticators方法，于是继续追踪，

```python
    def get_authenticators(self):
        """
        Instantiates and returns the list of authenticators that this view can use.
        """
        return [auth() for auth in self.authentication_classes]
```

可以看到是用列表生成式，去生成auth的对象，而auth类是循环authentication_classes的，而authentication_classes是从self自身开始查找的，那如何创建我们的认证类，就呼之欲出了。只需要在我们的视图类中创建这个属性就可以了。

## 自定义认证类

于是我们在IndexView类视图定义时指定

```python
class IndexView(APIView):
    authentication_classes = [MyAuthentication]
    def get(self, request, *args, **kwargs):
        ret = {
            "content": "index ok",
            "code": 200,
             "token": request.auth.token
        }
        return JsonResponse(ret)
```

然后在创建我们的认证类

```python
class MyAuthentication(object):
        # 自写的认证方法
    def authenticate(self, request):
        token =  request._request.GET.get("token")
        # 如果有token就直接验证token是否合法
        if token is not None:
            token_obj = TokenInfo.objects.filter(token=token).first()
            print(token_obj)
            if token_obj:
                # 如果合法就返回一个二元组 到时候其实是 request.user , request.auth 接收到的值
                return token_obj.user, token_obj
            else:
                raise exceptions.AuthenticationFailed("token is invaild!")
        # 没有登陆验证
        username = request._request.GET.get('username')
        password = request._request.GET.get('password')
        # token也没有 账号密码也获取不到 就直接抛出异常 认证失败
        if not username and not password:
            raise exceptions.AuthenticationFailed("认证失败！")

        user_obj = UserInfo.objects.filter(username=username, password=password).first()
        if user_obj:
            # 生成token 由密码和时间组合成的简单md5摘要算法  实际要更复杂点！！！
            token = hashlib.md5(bytes(str(time.time()) + password, encoding='utf-8')).hexdigest()
            # 如果没有这个用户就创建。如果有就更新token
            token_obj, created = TokenInfo.objects.update_or_create(
                user=user_obj,
                defaults={
                    'token': token,
                    'add_time': str(time.time())
                })
            return user_obj, token_obj
        raise exceptions.AuthenticationFailed("认证失败！")

    def authenticate_header(self, request):
        pass
```

这个认证类的逻辑自己制定，这个是我最终的代码。到这里并没有讲完，那怎么返回认证成功和失败值呢？

## 认证类返回值

可能乍一眼看到上面的认证类的返回内容会很迷糊，那来看看认证成功的返回值，在_authenticate方法中有

```python
            if user_auth_tuple is not None: # 如果上面认证方法返回的不是none
                self._authenticator = authenticator # 把认证的类给赋值给self
                self.user, self.auth = user_auth_tuple # 然后把认证返回后的元组 一个给user 一个给auth
                return
```

user_auth_tuple就是认证后返回的内容，下面user_auth_tuple会解包给user和auth的，那也就是说如果认证成功返回的值是要给他们的。所以就清楚了吧？什么？你说如果不成功呢？

不成功的话就会执行self._not_authenticated()方法（具体自己回看\_authenticate方法），来看这个函数干了什么坏事？

```python
    def _not_authenticated(self):
        """
        Set authenticator, user & authtoken representing an unauthenticated request.

        Defaults are None, AnonymousUser & None.
        """
        self._authenticator = None # 认证类为None

        if api_settings.UNAUTHENTICATED_USER:
            self.user = api_settings.UNAUTHENTICATED_USER()
        else:
            self.user = None

        if api_settings.UNAUTHENTICATED_TOKEN:
            self.auth = api_settings.UNAUTHENTICATED_TOKEN()
        else:
            self.auth = None
```

他在这边检测了是否有默认的值，如果没有就把auth和user设为none了

> PS：我自定义类中是直接抛出异常，这将会在try initial代码的时候被捕捉，是直接返回了！

![认证失败](https://upload-images.jianshu.io/upload_images/14657587-3e43c9739f1ef3bb.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![认证成功](https://upload-images.jianshu.io/upload_images/14657587-51f05ec25c3e3963.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![token认证](https://upload-images.jianshu.io/upload_images/14657587-3dca80309728b6dc.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


至此全部执行完代码后，一路返回到了initial这个原函数中执行perform_authentication的下一个check_permissions



# 权限permission

## 流程

有了认证authenticate的流程的熟悉，permission我们看起来简单很多

```python
    def check_permissions(self, request):
        """
        Check if the request should be permitted.
        Raises an appropriate exception if the request is not permitted.
        """
        for permission in self.get_permissions():
            if not permission.has_permission(request, self): # has_permission 权限认证方法
                # 如果到这里就是权限认证返回False 也就是没通过
                self.permission_denied( #
                    request, message=getattr(permission, 'message', None)
```

还是和authenticate一样的套路啊，还是get_permissions，但是这里是执行permission中的has_permission这个方法，那我们下面自己的定义的permission类中一定要有这个方法。下面我们先看get_permissions

```python
    def get_permissions(self):
        """
        Instantiates and returns the list of permissions that this view requires.
        """
        return [permission() for permission in self.permission_classes]
```

你看我说吧，一样的套路，那我们就直接可以在类视图IndexView中定义属性permission_class

```python
class IndexView(APIView):
    authentication_classes = [MyAuthentication]
    permission_classes = [MyPermission]
   
    def get(self, request, *args, **kwargs):
        ret = {
            "content": "index ok",
            "code": 200,
            "token": request.auth.token
        }
        return JsonResponse(ret)
```

## 自定义权限类

权限类我写的很简单，其中别忘了定义has_permission这个规定的方法。

```python
class MyPermission(object):
    """
    权限类
    """
    # 这个是认证失败返回的自定义消息
    message = '你只是普通用户！'

    def has_permission(self, request, view):

        ident = request.user
        if ident.role > 1:
            return True
        else:

            return False
```

其中的message定义是为了自定义错误信息的。因为你可以回去看check_permissions中他用反射取我们的message如果我们定义了就会用我们的。这个好像是hook吧？

## has_permission



### 参数

至于你问为什么has_permission要传这几个参数，你只能好好自己翻看他的调用过程和默认自带permission类了。我相信你会懂的

```python
class BasePermission(object):
    """
    A base class from which all permission classes should inherit.
    """

    def has_permission(self, request, view):
        """
        Return `True` if permission is granted, `False` otherwise.
        """
        return True

    def has_object_permission(self, request, view, obj):
        """
        Return `True` if permission is granted, `False` otherwise.
        """
        return True
```



## permission_denied

```python
    def permission_denied(self, request, message=None):
        """
        If request is not permitted, determine what kind of exception to raise.
        """
        if request.authenticators and not request.successful_authenticator:
            raise exceptions.NotAuthenticated()
        raise exceptions.PermissionDenied(detail=message)
```

如果返回False，也就是权限失败，会进入这个方法，这里还验证了下是否有认证类，然后选择到底抛出什么异常

![权限认证](https://upload-images.jianshu.io/upload_images/14657587-238788fc428bdee2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


至此我们，check_permissions也走完了，下面是check_throttles



# 频率throttles

我觉得这三个里面这个最有意思了，首先还是查看check_throttles这个代码

```python
    def check_throttles(self, request):
        """
        Check if request should be throttled.
        Raises an appropriate exception if the request is throttled.
        """
        for throttle in self.get_throttles():
            if not throttle.allow_request(request, self):
                self.throttled(request, throttle.wait())
```

还是用get_throttles方法去获取指定类，然后执行allow_request方法，那如果我们要自己写throttles你应该能猜到怎么办了把？

## 自定义访问控制类

```python
class MyThrottle(object):
    request_count = 3
    rate = 60 # 访问频率 60s
    time_temp = dict()

    def allow_request(self, request, view):
        remote_addr = request.META.get('REMOTE_ADDR')
        if remote_addr in self.time_temp.keys():
            # 已经访问过了 就代表在时间列表中了 那就要判断频率了
            time_list = self.time_temp.get(remote_addr)
            while time_list and time_list[-1] <= time.time() - self.rate:
                time_list.pop()
            if len(time_list) >= self.request_count:
                raise exceptions.Throttled()
            else:
                time_list.insert(0, time.time())
                self.time_temp[remote_addr] = time_list
                return True
        else:
            # 没有在键的列表中 肯定没有访问过 那是第一次肯定允许访问
            self.time_temp[remote_addr] = [time.time(),]
            return True



class IndexView(APIView):
    authentication_classes = [MyAuthentication]
    permission_classes = [MyPermission]
    throttle_classes = [MyThrottle]

    def get(self, request, *args, **kwargs):
        ret = {
            "content": "index ok",
            "code": 200,
            "token": request.auth.token
        }
        return JsonResponse(ret)
```

这里我定义的访问控制的逻辑是判断remote_addr，同一个ip只能在1分钟内访问3次否则不会返回正常内容

> PS：remote_addr一般不能伪造，除非好像听某表哥说在路由构造或许可以把

## throttled

我们来看看throttled是干什么的

```python
    def throttled(self, request, wait):
        """
        If request is throttled, determine what kind of exception to raise.
        """
        raise exceptions.Throttled(wait)
```

其实是来抛出异常的。而这个wait就是定义的消息内容，这个方式多种多样 这里不展开了。至此我们大致走了一遍过程了

![访问控制认证](https://upload-images.jianshu.io/upload_images/14657587-dbf933f707373b34.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


initial走完了，回到dispatch，下面就是正常的和View中一样的用反射找到对应方法。

# 示例的model结构

```python
from django.db import models

# Create your models here.


class UserInfo(models.Model):
    ROLE_CHOICE = {
        (1,"customer"),
        (2,"VIP"),
        (3,"SVIP")
    }
    username = models.CharField(max_length=20)
    password = models.CharField(max_length=256)
    role = models.IntegerField(choices=ROLE_CHOICE, default=1)


class TokenInfo(models.Model):
    user = models.OneToOneField(to=UserInfo, on_delete=models.CASCADE)
    token = models.CharField(max_length=64)
    add_time = models.CharField(max_length=50)
```

# 在setting中配置

前面都是在IndexView视图中指定类，其实可以配置在setting中的，rest_framework\view.py中给出了配置示例

```python
"""
Settings for REST framework are all namespaced in the REST_FRAMEWORK setting.
For example your project's `settings.py` file might look like this:

REST_FRAMEWORK = {
    'DEFAULT_RENDERER_CLASSES': (
        'rest_framework.renderers.JSONRenderer',
        'rest_framework.renderers.TemplateHTMLRenderer',
    )
    'DEFAULT_PARSER_CLASSES': (
        'rest_framework.parsers.JSONParser',
        'rest_framework.parsers.FormParser',
        'rest_framework.parsers.MultiPartParser'
    )
}

This module provides the `api_setting` object, that is used to access
REST framework settings, checking for user settings first, then falling
back to the defaults.
"""
```
