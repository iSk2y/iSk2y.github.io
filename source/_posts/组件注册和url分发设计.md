---
title: 组件注册和url分发设计
date: 2018.10.30 12:04
tags:
  - Django
  - Python
categories: Django
---

## 单例模式

**单例模式（Singleton Pattern）**是一种常用的软件设计模式，该模式的主要目的是确保**某一个类只有一个实例存在**。当你希望在整个系统中，某个类只能出现一个实例时，单例对象就能派上用场。

<!-- more -->
比如，某个服务器程序的配置信息存放在一个文件中，客户端通过一个 AppConfig 的类来读取配置文件的信息。如果在程序运行期间，有很多地方都需要使用配置文件的内容，也就是说，很多地方都需要创建 AppConfig 对象的实例，这就导致系统中存在多个 AppConfig 的实例对象，而这样会严重浪费内存资源，尤其是在配置文件内容很多的情况下。事实上，类似 AppConfig 这样的类，我们希望在程序运行期间只存在一个实例对象。

在 Python 中，我们可以用多种方法来实现单例模式：

- 使用模块
- 使用 `__new__`
- 使用装饰器（decorator）
- 使用元类（metaclass）



### 使用`__new__`

为了使类只能出现一个实例，我们可以使用 `__new__` 来控制实例的创建过程，代码如下：

```python
class Single(object):
    _instance = None
    def __new__(cls, *args, **kwargs):
        if not cls._instance:
            cls._instance = super(Single,cls).__new__(cls,*args,**kwargs)
        return cls._instance


class Myclass(Single):
    a = 1
```

在上面的代码中，我们将类的实例和一个类变量 `_instance` 关联起来，如果 `cls._instance` 为 None 则创建实例，否则直接返回 `cls._instance`。

```python

one = Myclass()
two = Myclass()

print(one == two)
print(one is two)
print(id(one),id(two))
---------------------------
True
True
2095469450464 2095469450464
```



### 使用模块

其实，**Python 的模块就是天然的单例模式**，因为模块在第一次导入时，会生成 `.pyc` 文件，当第二次导入时，就会直接加载 `.pyc` 文件，而不会再次执行模块代码。因此，我们只需把相关的**函数和数据定义在一个模块**中，就可以获得一个单例对象了。如果我们真的想要一个单例类，可以考虑这样做：

```python
# mysingleton.py
class My_Singleton(object):
    def foo(self):
        pass
 
my_singleton = My_Singleton()
```

将上面的代码保存在文件 `mysingleton.py` 中，然后在另一个脚本中这样引用测试：

```python
from mysingleton import my_singleton
print(id(my_singleton),my_singleton.a)
my_singleton.a = 5

from mysingleton import my_singleton
print(id(my_singleton),my_singleton.a)
-------------------------------------------
2639945371320 1
2639945371320 5

```

说明my_singleton是同一个对象，即单例模式



## admin执行流程

1. 循环加载执行所有已经注册的app中的admin.py文件

```python
def autodiscover():
    autodiscover_modules('admin', register_to=site)
```



2. 执行代码

```python
＃admin.py

class BookAdmin(admin.ModelAdmin):
    list_display = ("title",'publishDate', 'price')

admin.site.register(Book, BookAdmin) 
admin.site.register(Publish)
```



3. admin.site

![admin.site](http://upload-images.jianshu.io/upload_images/14657587-c8760ad1b2ee869b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里应用的是一个单例模式，对于AdminSite类的一个单例模式，执行的每一个app中的每一个admin.site都是一个对象



4. 执行register方法

```python

admin.site.register(Book, BookAdmin) 
admin.site.register(Publish)

class ModelAdmin(BaseModelAdmin):pass

def register(self, model_or_iterable, admin_class=None, **options):
    if not admin_class:
            admin_class = ModelAdmin
    # Instantiate the admin class to save in the registry
    self._registry[model] = admin_class(model, self)
```



> admin\site.py 部分源码

```python
    def register(self, model_or_iterable, admin_class=None, **options):
        
        if not admin_class:
            admin_class = ModelAdmin

        ………………

                # Instantiate the admin class to save in the registry
                self._registry[model] = admin_class(model, self)
```

如果没有指定admin_class，就默认使用ModelAdmin，然后最后把注册的model  **以model本身为键，admin_class为值**加入到_registry这个字典中（registry初始为空字典）

```python
admin.site.register(Book)
admin.site.register(Author)
admin.site.register(Publish)

print(admin.site._registry)

-------------------------------------------------------------------
{<class 'django.contrib.auth.models.Group'>: <django.contrib.auth.admin.GroupAdmin object at 0x0000018D47B4ACC0>, <class 'django.contrib.auth.models.User'>: <django.contrib.auth.admin.UserAdmin object at 0x0000018D47B75898>, <class 'copyadmin.models.Book'>: <django.contrib.admin.options.ModelAdmin object at 0x0000018D47B758D0>, <class 'copyadmin.models.Author'>: <django.contrib.admin.options.ModelAdmin object at 0x0000018D47B759E8>, <class 'copyadmin.models.Publish'>: <django.contrib.admin.options.ModelAdmin object at 0x0000018D47B75A58>}


```

到这里admin的注册流程结束！



5. admin的url配置

```python
urlpatterns = [
    url(r'^admin/', admin.site.urls),
]
```

```python
class AdminSite(object):
    
     def get_urls(self):
        from django.conf.urls import url, include
      
        urlpatterns = []

        # Add in each model's views, and create a list of valid URLS for the
        # app_index
        valid_app_labels = []
        
        # 循环self._registry中的每个项，得到他们的app名称字符串和model名称字符串
        for model, model_admin in self._registry.items():
            urlpatterns += [
                url(r'^%s/%s/' % (model._meta.app_label, model._meta.model_name), include(model_admin.urls)),
            ]
            if model._meta.app_label not in valid_app_labels:
                valid_app_labels.append(model._meta.app_label)

      
        return urlpatterns

    @property
    def urls(self):
        return self.get_urls(), 'admin', self.name
```

主要利用url分发的方式。动态构造分发的url列表

循环`self._registry `也就是`admin.site._registry`列表中的每个项，构造出app名称+model名称url



> url方法拓展，简单分发

```python
from django.shortcuts import HttpResponse
def test01(request):
    return HttpResponse("test01")

def test02(request):
    return HttpResponse("test02")

urlpatterns = [
    url(r'^admin/', admin.site.urls),
    url(r'^yuan/', ([
                    url(r'^test01/', test01),
                    url(r'^test02/', test02),

                    ],None,None)),

]
```



## 模拟admin方式设计url

```python
def show(request):
    return HttpResponse("show")


def add(request):
    return HttpResponse("add")


def delete(request):
    return HttpResponse("delete")


def change(request):
    return HttpResponse("change")


def get_action_urls():
    action_url = [
        url(r'^$',show),
        url(r'^add/',add),
        url(r'^del/',delete),
        url(r'^change/',change)
    ]
    return action_url, None, None


def get_urls():
    tempurl = []
    for model,model_admin in admin.site._registry.items():
        tempurl.append(url(r'^{}/{}/'.format(model._meta.app_label,model._meta.model_name),get_action_urls()))
    return tempurl,None,None



urlpatterns = [
    url(r'^admin/', admin.site.urls),
    url(r'^test/', get_urls()),
]
```

构成的url

```
^admin/
^test/ ^auth/group/ ^add/
^test/ ^auth/group/ ^del/
^test/ ^auth/group/ ^change/
^test/ ^auth/user/
^test/ ^auth/user/ ^add/
^test/ ^auth/user/ ^del/
^test/ ^auth/user/ ^change/
^test/ ^copyadmin/book/
^test/ ^copyadmin/book/ ^add/
^test/ ^copyadmin/book/ ^del/
^test/ ^copyadmin/book/ ^change/
^test/ ^copyadmin/author/
^test/ ^copyadmin/author/ ^add/
^test/ ^copyadmin/author/ ^del/
^test/ ^copyadmin/author/ ^change/
^test/ ^copyadmin/publish/
^test/ ^copyadmin/publish/ ^add/
^test/ ^copyadmin/publish/ ^del/
^test/ ^copyadmin/publish/ ^change/
```


## 组件对应设计


demo项目文件目录格式

```
modelDemo/
├── app01
│   ├── __init__.py
│   ├── __pycache__
│   ├── admin.py
│   ├── apps.py
│   ├── migrations
│   │   ├── 0001_initial.py
│   │   ├── __init__.py
│   │   └── __pycache__
│   ├── models.py
│   ├── tests.py
│   ├── views.py
│   └── yadmin.py
├── app02
│   ├── __init__.py
│   ├── __pycache__
│   ├── admin.py
│   ├── apps.py
│   ├── migrations
│   │   ├── __init__.py
│   │   └── __pycache__
│   ├── models.py
│   ├── tests.py
│   ├── views.py
│   └── yadmin.py
├── db.sqlite3
├── manage.py
├── modelDemo
│   ├── __init__.py
│   ├── __pycache__
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── templates
└── yadmin
    ├── __init__.py
    ├── __pycache__
    ├── admin.py
    ├── apps.py
    ├── migrations
    │   ├── __init__.py
    │   └── __pycache__
    ├── models.py
    ├── service
    │   ├── __init__.py
    │   ├── __pycache__
    │   └── yadmin.py
    ├── tests.py
    └── views.py
```



> yadmin / service / yadmin.py 

```python


from django.http import HttpResponse
from django.conf.urls import url



class Modelyadmin(object):
    def __init__(self,model,site):
        self.model = model
        self.site = site
        
    def list_view(self,request):
        return HttpResponse("list")   

    def add_view(self,request):
        return HttpResponse("add")

    def delete_view(self,request):
        return HttpResponse("delete")

    def change_view(self,request):
        return HttpResponse("change")

    def get_action_urls(self):
        action_url = [
            url(r'^$', self.list_view),
            url(r'^add/', self.add_view),
            url(r'^del/', self.delete_view),
            url(r'^change/', self.change_view)
        ]
        return action_url

    @property
    def action_urls(self):
        return self.get_action_urls(),None,None


class yadminSite(object):
    def __init__(self,name='admin'):
        self._registry = {}

    def register(self,model,admin_class=None,**options):
        if not admin_class:
            admin_class = Modelyadmin

        self._registry[model] = admin_class(model,self)

    def get_urls(self):
        tempurl = []

        for model, model_admin in self._registry.items():
            app_label = model._meta.app_label # app的名称 str类型
            model_name = model._meta.model_name # 模型名称 str类型
            tempurl.append(url(r'^{0}/{1}/'.format(app_label, model_name), model_admin.action_urls))
             # 这边要用model_admin.action_urls二级分发url因为要根据不同配置类显示不同想要的功能
        return tempurl

    @property
    def urls(self):
        return self.get_urls(),None,None


site = yadminSite()
```

简单设计注册组件和url分发设计



> yadmin / apps.py

```python
from django.apps import AppConfig
from django.utils.module_loading import autodiscover_modules


class YadminConfig(AppConfig):
    name = 'yadmin'
    
	#ready函数 在启动yadmin这个app的时候 会自动运行这个函数，
    # 在该函数中 设置自动发现运行每个app中的yadmin.py文件
    def ready(self):
        autodiscover_modules('yadmin')
```