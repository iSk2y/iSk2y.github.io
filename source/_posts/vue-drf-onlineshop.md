---
title: Vue+drf前后端分离之生鲜电商项目(一)
date: 2018-11-28 10:00:13
tags:
  - Django
  - Python
  - Vue
categories: Django
---

花了一周多时间了解了Django restframework和Vue2.0+Vue Router+Vuex全家桶之后，选择了一个项目来了解下这类前后端项目的开发流程。故此笔记由来。


<!-- more -->

# Github仓库

代码同步在GitHub中，需要自取，重大功能完成commit了。
https://github.com/iSk2y/vueshop


# 开发环境

- Python 3.6.5
- Django 2.1
- OS：Windows 10
- Django restframework 3.9
- Vue 2.9.6
- MySQL 5.7

其他的依赖请看requirements



# 目录结构

```
OnlineShop/
├── apps
│   ├── goods
│   ├── __init__.py
│   ├── trade
│   ├── user_operation
│   └── users
├── db_tools
├── extra_apps
│   └── __init__.py
├── manage.py
├── media
├── OnlineShop
│   ├── __init__.py
│   ├── __pycache__
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── README.MD
└── templates
```

项目初始目录结构，后续可能有变动



创建三个app

- goods        商品
- trade          交易
- user_operation       用户操作

# Models

## users models

> users/models.py

```python
import datetime

from django.db import models
from django.contrib.auth.models import AbstractUser

# Create your models here.


class UserProfile(AbstractUser):
    """
    自定义用户表
    """
    GENDER_CHOICES = (
        ("male", "男"),
        ("female", "女")
    )
    # 用户用手机注册，所以姓名，生日和邮箱可以为空
    name = models.CharField("姓名", max_length=30, null=True, blank=True)
    birthday = models.DateField("出生年月", null=True, blank=True)
    gender = models.CharField("性别", max_length=6, choices=GENDER_CHOICES, default="female")
    mobile = models.CharField("电话", max_length=11)
    email = models.EmailField("邮箱", max_length=100, null=True, blank=True)

    class Meta:
        verbose_name = '用户'
        verbose_name_plural = verbose_name

    def __str__(self):
        return self.name


class VerifyCode(models.Model):
    """
    验证码
    """
    code = models.CharField("验证码", max_length=10)
    mobile = models.CharField("电话", max_length=11)
    add_time = models.DateTimeField("添加时间", default= datetime.now)

    class Meta:
        verbose_name = "短信验证"
        verbose_name_plural = verbose_name

    def __str__(self):
        return self.code
```

重载了系统auth用户表，要在settings中声明

```python
#重载系统的用户，让UserProfile生效
AUTH_USER_MODEL = 'users.UserProfile'
```





## goods models

```python
import datetime

from django.db import models
from DjangoUeditor.models import UEditorField
# Create your models here.


class GoodsCategory(models.Model):
    """
    商品分类
    """
    CATEGORY_TYPE = (
        (1, "一级类目"),
        (2, "二级类目"),
        (3, "三级类目"),
    )

    name = models.CharField('类别名', default="", max_length=30, help_text="类别名")
    code = models.CharField("类别code", default="", max_length=30, help_text="类别code")
    desc = models.TextField("类别描述", default="", help_text="类别描述")
    # 目录树级别
    category_type = models.IntegerField("类目级别", choices=CATEGORY_TYPE, help_text="类目级别")
    # 设置models有一个指向自己的外键
    parent_category = models.ForeignKey("self", on_delete=models.CASCADE, null=True, blank=True, verbose_name="父类目级别",
                                        help_text="父目录", related_name="sub_cat")
    is_tab = models.BooleanField("是否导航", default=False, help_text="是否导航")
    add_time = models.DateTimeField("添加时间", default=datetime.now)

    class Meta:
        verbose_name = "商品类别"
        verbose_name_plural = verbose_name

    def __str__(self):
        return self.name


class GoodsCategoryBrand(models.Model):
    """
    某一大类下的宣传商标
    """
    category = models.ForeignKey(GoodsCategory, on_delete=models.CASCADE, related_name='brands', null=True, blank=True, verbose_name="商品类目")
    name = models.CharField("品牌名", default="", max_length=30, help_text="品牌名")
    desc = models.TextField("品牌描述", default="", max_length=200, help_text="品牌描述")
    image = models.ImageField(max_length=200, upload_to="brands/")
    add_time = models.DateTimeField("添加时间",default=datetime.now)

    class Meta:
        verbose_name = "宣传品牌"
        verbose_name_plural = verbose_name
        db_table = "goods_goodsbrand"

    def __str__(self):
        return self.name


class Goods(models.Model):
    """
    商品
    """
    category = models.ForeignKey(GoodsCategory, on_delete=models.CASCADE, verbose_name="商品类目")
    goods_sn = models.CharField("商品唯一货号", max_length=50, default="")
    name = models.CharField("商品名", max_length=100,)
    click_num = models.IntegerField("点击数", default=0)
    sold_num = models.IntegerField("商品销售量",default=0)
    fav_num = models.IntegerField("收藏数", default=0)
    goods_num = models.IntegerField("库存数", default=0)
    market_price = models.FloatField("市场价格", default=0)
    shop_price = models.FloatField("本店价格", default=0)
    goods_brief = models.TextField("商品简短描述", max_length=500)
    goods_desc = UEditorField(verbose_name=u"内容", imagePath="goods/images/", width=1000, height=300,
                              filePath="goods/files/", default='')
    ship_free = models.BooleanField("是否承担运费", default=True)
    # 首页中展示的商品封面图
    goods_front_image = models.ImageField(upload_to="goods/images/", null=True, blank=True, verbose_name="封面图")
    # 首页中新品展示
    is_new = models.BooleanField("是否新品", default=False)
    # 商品详情页的热卖商品，自行设置
    is_hot = models.BooleanField("是否热销", default=False)
    add_time = models.DateTimeField("添加时间", default=datetime.now)

    class Meta:
        verbose_name = '商品信息'
        verbose_name_plural = verbose_name

    def __str__(self):
        return self.name


class GoodsImage(models.Model):
    """
    商品轮播图
    """
    goods = models.ForeignKey(Goods, on_delete=models.CASCADE, verbose_name="商品", related_name="images")
    image = models.ImageField(upload_to="", verbose_name="图片", null=True, blank=True)
    add_time = models.DateTimeField("添加时间", default=datetime.now)

    class Meta:
        verbose_name = '商品轮播'
        verbose_name_plural = verbose_name

    def __str__(self):
        return self.goods.name


class Banner(models.Model):
    """
    首页轮播的商品
    """
    goods = models.ForeignKey(Goods, on_delete=models.CASCADE, verbose_name="商品")
    image = models.ImageField(upload_to='banner', verbose_name="轮播图片")
    index = models.IntegerField("轮播顺序",default=0)
    add_time = models.DateTimeField("添加时间", default=datetime.now)

    class Meta:
        verbose_name = '首页轮播'
        verbose_name_plural = verbose_name

    def __str__(self):
        return self.goods.name


class IndexAd(models.Model):
    """
    商品广告
    """
    category = models.ForeignKey(GoodsCategory, on_delete=models.CASCADE, related_name='category',verbose_name="商品类目")
    goods = models.ForeignKey(Goods, on_delete=models.CASCADE, related_name='goods')

    class Meta:
        verbose_name = '首页广告'
        verbose_name_plural = verbose_name

    def __str__(self):
        return self.goods.name


class HotSearchWords(models.Model):
    """
    搜索栏下方热搜词
    """
    keywords = models.CharField("热搜词",default="", max_length=20)
    index = models.IntegerField("排序",default=0)
    add_time = models.DateTimeField("添加时间", default=datetime.now)

    class Meta:
        verbose_name = '热搜排行'
        verbose_name_plural = verbose_name

    def __str__(self):
        return self.keywords

```



## trade models

```python
from datetime import datetime

from django.db import models
from goods.models import Goods
from users.models import UserProfile
# Create your models here.

# get_user_model方法会去setting中找AUTH_USER_MODEL
from django.contrib.auth import get_user_model
User = get_user_model()


# Create your models here.
class ShoppingCart(models.Model):
    """
    购物车
    """
    user = models.ForeignKey(User, on_delete=models.CASCADE, verbose_name="用户")
    goods = models.ForeignKey(Goods, on_delete=models.CASCADE, verbose_name="商品")
    nums = models.IntegerField("购买数量",default=0)

    add_time = models.DateTimeField(default=datetime.now, verbose_name="添加时间")

    class Meta:
        verbose_name = '购物车喵'
        verbose_name_plural = verbose_name
        unique_together = ("user", "goods")

    def __str__(self):
        return "%s(%d)".format(self.goods.name, self.nums)


class OrderInfo(models.Model):
    """
    订单信息
    """
    ORDER_STATUS = (
        ("TRADE_SUCCESS", "成功"),
        ("TRADE_CLOSED", "超时关闭"),
        ("WAIT_BUYER_PAY", "交易创建"),
        ("TRADE_FINISHED", "交易结束"),
        ("paying", "待支付"),
    )
    PAY_TYPE = (
        ("alipay", "支付宝"),
        ("wechat", "微信"),
    )

    user = models.ForeignKey(User, on_delete=models.CASCADE, verbose_name="用户")
    # 订单号唯一
    order_sn = models.CharField("订单编号", max_length=30, null=True, blank=True, unique=True)
    # 微信支付会用到
    nonce_str = models.CharField("随机加密串", max_length=50, null=True, blank=True, unique=True)
    # 支付宝交易号
    trade_no = models.CharField("交易号", max_length=100, unique=True, null=True, blank=True)
    # 支付状态
    pay_status = models.CharField("订单状态", choices=ORDER_STATUS, default="paying", max_length=30)
    # 订单的支付类型
    pay_type = models.CharField("支付类型", choices=PAY_TYPE, default="alipay", max_length=10)
    post_script = models.CharField("订单留言", max_length=200)
    order_mount = models.FloatField("订单金额", default=0.0)
    pay_time = models.DateTimeField("支付时间", null=True, blank=True)

    # 用户信息
    address = models.CharField("收货地址", max_length=100, default="")
    signer_name = models.CharField("签收人", max_length=20, default="")
    singer_mobile = models.CharField("联系电话", max_length=11)

    add_time = models.DateTimeField("添加时间",default=datetime.now)

    class Meta:
        verbose_name = "订单信息"
        verbose_name_plural = verbose_name

    def __str__(self):
        return str(self.order_sn)


class OrderGoods(models.Model):
    """
    订单内的商品详情
    """
    # 一个订单对应多个商品
    order = models.ForeignKey(OrderInfo, on_delete=models.CASCADE, verbose_name="订单信息", related_name="goods")
    # 两个外键形成一张关联表
    goods = models.ForeignKey(Goods, on_delete=models.CASCADE, verbose_name="商品")
    goods_num = models.IntegerField("商品数量", default=0)

    add_time = models.DateTimeField("添加时间", default=datetime.now)

    class Meta:
        verbose_name = "订单商品"
        verbose_name_plural = verbose_name

    def __str__(self):
        return str(self.order.order_sn)

```



## user_operation

```python
from datetime import datetime
from django.db import models
from goods.models import Goods

from django.contrib.auth import get_user_model
User = get_user_model()


class UserFav(models.Model):
    """
    用户收藏操作
    """
    user = models.ForeignKey(User, on_delete=models.CASCADE, verbose_name="用户")
    goods = models.ForeignKey(Goods, on_delete=models.CASCADE, verbose_name="商品", help_text="商品id")
    add_time = models.DateTimeField("添加时间",default=datetime.now)

    class Meta:
        verbose_name = '用户收藏'
        verbose_name_plural = verbose_name
        unique_together = ("user", "goods")

    def __str__(self):
        return self.user.username


class UserAddress(models.Model):
    """
    用户收货地址
    """
    user = models.ForeignKey(User, on_delete=models.CASCADE, verbose_name="用户" )
    province = models.CharField("省份",max_length=100, default="")
    city = models.CharField("城市",max_length=100, default="")
    district = models.CharField("区域",max_length=100, default="")
    address = models.CharField("详细地址",max_length=100, default="")
    signer_name = models.CharField("签收人",max_length=100, default="")
    signer_mobile = models.CharField("电话",max_length=11, default="")
    add_time = models.DateTimeField("添加时间",default=datetime.now)

    class Meta:
        verbose_name = "收货地址"
        verbose_name_plural = verbose_name

    def __str__(self):
        return self.address


class UserLeavingMessage(models.Model):
    """
    用户留言
    """
    MESSAGE_CHOICES = (
        (1, "留言"),
        (2, "投诉"),
        (3, "询问"),
        (4, "售后"),
        (5, "求购")
    )
    user = models.ForeignKey(User, on_delete=models.CASCADE, verbose_name="用户")
    message_type = models.IntegerField(default=1, choices=MESSAGE_CHOICES, verbose_name="留言类型",
                                      help_text=u"留言类型: 1(留言),2(投诉),3(询问),4(售后),5(求购)")
    subject = models.CharField("主题",max_length=100, default="")
    message = models.TextField("留言内容",default="",help_text="留言内容")
    file = models.FileField(upload_to="message/images/", verbose_name="上传的文件", help_text="上传的文件")
    add_time = models.DateTimeField("添加时间",default=datetime.now)

    class Meta:
        verbose_name = "用户留言"
        verbose_name_plural = verbose_name

    def __str__(self):
        return self.subject
```

# 安装xadmin

[Xadmin for Django2.0](https://github.com/sshwsfc/xadmin/tree/django2)

```
pip install https://codeload.github.com/sshwsfc/xadmin/zip/django2
```

速度可能有点慢，可以手动下载那个zip，然后pip install

- 安装完，记得注册进APP然后修改URL路由，再migrate一下生成表
- 记得在APP中新建adminx.py，这样xadmin才会自动发现到。

如果出现错误如：

```
Model class django.contrib.admin.models.LogEntry doesn't declare an explicit app_label and isn't in an application in INSTALLED_APPS.
```

在setting中查看是否注册了admin这个app

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'xadmin',
    'crispy_forms'
]
```

这些是xadmin必须要注册的



# 其他package安装

- pip install coreapi                         drf的文档支持
- pip install django-guardian           drf对象级别的权限支持



# 跨域问题解决

曾经自己了解过跨域问题 可以在 [跨域请求Jsonp和CORS](http://isk2y.coding.me/2018/10/28/%E8%B7%A8%E5%9F%9F%E8%AF%B7%E6%B1%82Jsonp%E5%92%8CCORS/) 看到。解决方法：

1. jsonp
2. cors

现在大多是采用后者来解决了，更符合标准和规范。

## 解决方法

既然是采用CORS方式来解决跨域，那么只要在response的header中加入相应内容就可以了，在django中也有对应插件

> django-cors-headers模块：https://github.com/ottoyiu/django-cors-headers

1. 安装

```
pip install django-cors-headers
```

2. 添加到INSTALL_APPS中

```python
INSTALLED_APPS = (
    ...
    'coreschema',
 ... )
```

3. 添加到中间件，尽可能放前面，**必须在CsrfViewMiddleware之前**

```python
MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware', //可以放第一个
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
```

4. 设置为True

```python
CORS_ORIGIN_ALLOW_ALL = True
```







# 创建API

## 商品分类api

商品分类有两个接口：

1. 一种是全部分类：一级二级三级

![分类](http://upload-images.jianshu.io/upload_images/14657587-9a37b47a371cca17.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




2. 某一类的分类以及商品详细信息：

![商品](http://upload-images.jianshu.io/upload_images/14657587-c101f879f3837c75.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




### 序列化类

查看前端Vue源码中的数据操作，

```html
<li class="first" v-for="(item,index) in allMenuLabel" @mouseover="overChildrenmenu(index)" @mouseout="outChildrenmenu(index)">
                              <h3 style="background:url(../images/1449088788518670880.png) 20px center no-repeat;">
                                <router-link :to="'/app/home/list/'+item.id">{{item.name}}</router-link> </h3>
                                <div class="J_subCata" id="J_subCata" v-show="showChildrenMenu ===index"  style=" left: 215px; top: 0px;">
                                    <div class="J_subView" >
                                      <div v-for="list in item.sub_cat">
                                        <dl>
                                          <dt>
                                            <router-link :to="'/app/home/list/'+list.id">{{list.name}}</router-link>
                                          </dt>

                                          <dd>
                                            <router-link  v-for="childrenList in list.sub_cat" :key="childrenList.id" :to="'/app/home/list/'+childrenList.id">{{childrenList.name}}</router-link>
                                          </dd>
                                        </dl>
                                        <div class="clear"></div>
                                      </div>
                                    </div>
                                </div>
                            </li>
```

看来sub_cat也就是子类是不可少的字段。视频中使用了显示指定字段 来设置sub_cat循环嵌套

```python

from rest_framework import serializers
from .models import Goods,GoodsCategory


class CategorySerializer3(serializers.ModelSerializer):
    '''三级分类'''
    class Meta:
        model = GoodsCategory
        fields = "__all__"


class CategorySerializer2(serializers.ModelSerializer):
    '''
    二级分类
    '''
    #在parent_category字段中定义的related_name="sub_cat" 
    sub_cat = CategorySerializer3(many=True)
    class Meta:
        model = GoodsCategory
        fields = "__all__"


class CategorySerializer(serializers.ModelSerializer):
    """
    商品一级类别序列化
    """
    sub_cat = CategorySerializer2(many=True)
    class Meta:
        model = GoodsCategory
        fields = "__all__"
```

这样目的就是嵌套序列化，如果有子类继续序列化。我记得django restframework中有个depth可以控制深度，于是我写成了这样

```python
class CategorySerializer(serializers.ModelSerializer):
    """
    商品类别序列化
    """

    class Meta:
        model = GoodsCategory
        fields = ("id", "name", "code", "desc", "category_type", "is_tab", "add_time", "sub_cat")
        depth = 2
```

这里fields只能显示指定了，不能直接用all，那样无法取到sub_cat（sub_cat是related_name）然后我把parent_category去掉了。但是后来想到，depth二次嵌套后 序列化出来的对象的字段只是model对应字段也就是说无法再次查询sub_cat了。emmm ，当然我们也可以采用filter type=3的然后将parent category作为sub_cat就是倒置一下，但是得配合前端修改内容，所以决定还是使用视频中的写法。

### view

```python
class CategoryViewSet(mixins.ListModelMixin, mixins.RetrieveModelMixin, viewsets.GenericViewSet):
    """
    list:
        商品分类列表数据
    """

    queryset = GoodsCategory.objects.filter(category_type=1)
    serializer_class = CategorySerializer
    pagination_class = None
```



### url配置

用drf的路由来自动配置

```python

router = routers.DefaultRouter()
router.register(r'categorys',views.CategoryViewSet, base_name="categorys")

urlpatterns = [
   
    path('', include(router.urls)),
]
```





## 商品列表

找到对应Vue的源码中商品列表的数据来源是这样：

```javascript
getListData() {
                if(this.pageType=='search'){
                  getGoods({
                    search: this.searchWord, //搜索关键词
                  }).then((response)=> {
                    this.listData = response.data.results;
                    this.proNum = response.data.count;
                  }).catch(function (error) {
                    console.log(error);
                  });
                }else {
                  getGoods({
                    page: this.curPage, //当前页码
                    top_category: this.top_category, //商品类型
                    ordering: this.ordering, //排序类型
                    pricemin: this.pricemin, //价格最低 默认为‘’ 即为不选价格区间
                    pricemax: this.pricemax // 价格最高 默认为‘’
                  }).then((response)=> {

                    this.listData = response.data.results;
                    this.proNum = response.data.count;
                  }).catch(function (error) {
                    console.log(error);
                  });
                }

```

查看getGoods这个api

```javascript
//获取商品列表
export const getGoods = params => { return axios.get(`${host}/goods/`, { params: params }) }
```



api要和前端筛选的字段相同，这里要引入过滤器



### 过滤器

drf的filter用法   [http://www.django-rest-framework.org/api-guide/filtering/](https://link.jianshu.com/?t=http%3A%2F%2Fwww.django-rest-framework.org%2Fapi-guide%2Ffiltering%2F)

```python
# goods/filters.py

import django_filters

from .models import Goods
from django.db.models import Q

class GoodsFilter(django_filters.rest_framework.FilterSet):
    """
    商品过滤的类
    """
    # 两个参数，name是要过滤的字段，lookup是执行的行为，‘小与等于本店价格’
    # 两个参数，name是要过滤的字段，lookup是执行的行为，‘小与等于本店价格’
    pricemin = django_filters.NumberFilter(field_name="shop_price", lookup_expr='gte')
    pricemax = django_filters.NumberFilter(field_name="shop_price", lookup_expr='lte')
    top_category = django_filters.NumberFilter(field_name="category", method='top_category_filter')

    def top_category_filter(self, queryset, name, value):
        # 不管当前点击的是一级分类二级分类还是三级分类，都能找到。
        return queryset.filter(Q(category_id=value) | Q(category__parent_category_id=value) | Q(
            category__parent_category__parent_category_id=value))

    class Meta:
        model = Goods
        fields = ['pricemin', 'pricemax']
```

### view

```python
class GoodsPagination(PageNumberPagination):
    """
    商品列表自定义分页
    """
    # 默认每页显示的个数
    page_size = 12
    # 可以动态改变每页显示的个数
    page_size_query_param = 'page_size'
    # 页码参数
    page_query_param = 'page'
    # 最多能显示多少页
    max_page_size = 100
    

class GoodsListViewSet(mixins.ListModelMixin, viewsets.GenericViewSet):
    """
    list:
        商品列表数据
    """
    queryset = Goods.objects.all().order_by("id")
    pagination_class = GoodsPagination
    # 配置filter的类 
    filter_backends = (DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter)
    filter_class = GoodsFilter
    # 设置filter的类为我们自定义的类
    serializer_class = GoodSerializer

    # 搜索
    search_fields = ('name', 'goods_brief', 'goods_desc')
    # 排序
    ordering_fields = ('sold_num', 'shop_price')
```



### url配置

```python
router.register(r'goods', views.GoodsListViewSet, base_name='goods')
```



# 登陆功能

## json web token完成认证

介绍：http://getblimp.github.io/django-rest-framework-jwt/

1. 在settings.py中配置下

```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework.authentication.BasicAuthentication',
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework_jwt.authentication.JSONWebTokenAuthentication',
    )
}
```



2. urls中配置

```python
   from rest_framework_jwt.views import refresh_jwt_token
    #  ...

    urlpatterns = [
        #  ...
        path('jwt-auth/', obtain_jwt_token )
    ]
```

![jwt](https://upload-images.jianshu.io/upload_images/14657587-890cbede395cbefa.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




## 和前端vue做接口调试

前端中的api如下

```javascript
// api.js
//登录
export const login = params => {
  return axios.post(`${host}/login/`, params)
}
```

在login.vue中真正的操作如下

```javascript
login({
          username:this.userName, //当前页码
          password:this.parseWord
      }).then((response)=> {
            console.log(response);
            //本地存储用户信息
            cookie.setCookie('name',this.userName,7);
            cookie.setCookie('token',response.data.token,7)
            //存储在store
            // 更新store数据
            that.$store.dispatch('setInfo');
            //跳转到首页页面
            this.$router.push({ name: 'index'})
          })
          .catch(function (error) {
            if("non_field_errors" in error){
              that.error = error.non_field_errors[0];
            }
            if("username" in error){
              that.userNameError = error.username[0];
            }
            if("password" in error){
              that.parseWordError = error.password[0];
            }
          });
```

axios中配置了request的拦截器，用来配置jwt



```javascript
// axios\index.js
// http request 拦截器
axios.interceptors.request.use(
  config => {
    if (store.state.userInfo.token) {  // 判断是否存在token，如果存在的话，则每个http header都加上token
      config.headers.Authorization = `JWT ${store.state.userInfo.token}`;
    }
    return config;
  },
  err => {
    return Promise.reject(err);
  });
```



为了前后端保持一直

1. 后端的url改成login
2. 前端可以用手机号码登陆，所以修改后端auth认证的逻辑



### 自定义用户认证

默认是用django自己的auth，也就是用户表的登陆验证，我们要指定并重载

```python
# users.views.py

from django.contrib.auth.backends import ModelBackend
from django.contrib.auth import get_user_model
from django.db.models import Q

User = get_user_model()

class CustomBackend(ModelBackend):
    """
    自定义用户验证
    """
    def authenticate(self, request, username=None, password=None, **kwargs):
        try:
            # 用户名和手机都能登录
            user = User.objects.get(
                Q(username=username) | Q(mobile=username))
            if user.check_password(password):
                return user
        except Exception as e:
            return None
```

然后配置到settings中去

```python
AUTHENTICATION_BACKENDS = (
    'users.views.CustomBackend',
)
```





## JWT的时效时间设置

jwt其实还有很多设置选项，配置在setting中

```python
import datetime
#有效期限
JWT_AUTH = {
    'JWT_EXPIRATION_DELTA': datetime.timedelta(days=7),    #也可以设置seconds=20
    'JWT_AUTH_HEADER_PREFIX': 'JWT',                       #JWT跟前端保持一致，比如“token”这里设置成JWT
}
```



# 注册功能



## 云片网验证码

利用云片网发送验证码来注册账号。因为是练习，项目不实际上线，此处不做实际测试

```python
#!/usr/bin/env python
# -*- coding:utf-8 -*-
#    @Author:iSk2y

import requests
import json


class YunPian(object):

    def __init__(self, api_key):
        self.api_key = api_key
        self.single_send_url = "https://sms.yunpian.com/v2/sms/single_send.json"

    def send_sms(self, code, mobile):
        """
        实际发送短信方法
        :param code: 随机code验证码
        :param mobile: 手机号码
        :return: 云片网接口返回内容 dict
        """

        # 需要传递的参数
        params = {
            "apikey": self.api_key,
            "mobile": mobile,
            "text": "【慕雪生鲜超市】您的验证码是{code}。如非本人操作，请忽略本短信".format(code=code)
        }
        
        response = requests.post(self.single_send_url, data=params)
        re_dict = json.loads(response.text)
        return re_dict
```



## 注册逻辑验证

1. 手机号码是否合法
2. 手机号码是否已经被注册
3. 发送验证码频率验证

```python
# users/serializers.py
#!/usr/bin/env python
# -*- coding:utf-8 -*-
#    @Author:iSk2y
import re
from datetime import datetime, timedelta
from rest_framework import serializers
from users.models import VerifyCode
from django.contrib.auth import get_user_model
User = get_user_model()


# 手机号码正则表达式
REGEX_MOBILE = "^1[358]\d{9}$|^147\d{8}$|^176\d{8}$"


class SmsSerializer(serializers.Serializer):
    """
    注册提交手机序列化类
    """
    mobile = serializers.CharField(max_length=11)

    def validate_mobile(self, mobile):
        """
        自定义验证手机号码
        :return:
        """
        # 是否已经注册
        # 是否已经注册
        if User.objects.filter(mobile=mobile).count():
            raise serializers.ValidationError("用户已经存在")

        # 是否合法
        if not re.match(REGEX_MOBILE, mobile):
            raise serializers.ValidationError("手机号码非法")

        # 验证码发送频率
        # 60s内只能发送一次
        one_mintes_ago = datetime.now() - timedelta(hours=0, minutes=1, seconds=0)
        if VerifyCode.objects.filter(add_time__gt=one_mintes_ago, mobile=mobile).count():
            raise serializers.ValidationError("距离上一次发送未超过60s")

        return mobile
```

上面是序列化类，下面是视图逻辑

```python
# users/views.py
class SmsCodeViewset(CreateModelMixin,viewsets.GenericViewSet):
    """
    手机验证码
    """
    serializer_class = SmsSerializer

    def generate_code(self):
        """
        生成四位数字的验证码
        """
        seeds = "1234567890"
        random_str = []
        for i in range(4):
            random_str.append(choice(seeds))

        return "".join(random_str)

    def create(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        # 验证合法
        serializer.is_valid(raise_exception=True)

        mobile = serializer.validated_data["mobile"]

        yun_pian = YunPian(APIKEY)
        # 生成验证码
        code = self.generate_code()

        sms_status = yun_pian.send_sms(code=code, mobile=mobile)

        if sms_status["code"] != 0:
            # 发送失败了 
            return Response({
                "mobile": sms_status["msg"]
            }, status=status.HTTP_400_BAD_REQUEST)
        else:
            code_record = VerifyCode(code=code, mobile=mobile)
            code_record.save()
            return Response({
                "mobile": mobile
            }, status=status.HTTP_201_CREATED)
```

这是发短信验证码前的逻辑



然后输入验证码后的逻辑判断：

1. 验证码发送了多个取最后一个
2. 验证码过期
3. 验证码错误



用户注册序列化类

```python
# users/serializers.py
class UserRegSerializer(serializers.ModelSerializer):
    """
    用户注册序列化类
    """
    # UserProfile中没有code字段， 这里需要对应自定义一个code字段 否则会失败
    code = serializers.CharField(
        max_length=4, min_length=4, required=True, write_only=True,
         error_messages={
             "blank": "请输入验证码",
             "required": "请输入验证码",
             "max_length": "验证码格式错误",
             "min_length": "验证码格式错误"
         }, help_text="验证码"
    )
    # 验证用户名是否存在
    username = serializers.CharField(label='用户名', help_text='用户名', required=True, allow_blank=False,
                                     validators=[UniqueValidator(queryset=User.objects.all(), message="用户已经存在")])

    password = serializers.CharField(style={'input_type': 'password'}, write_only=True, required=True, label='密码')

    # 密码加密保存
    def create(self, validated_data):
        user = super(UserRegSerializer, self).create(validated_data=validated_data)
        user.set_password(validated_data["password"])
        user.save()
        return user

    # 验证code
    def validate_code(self, code):
        # 用户注册，已post方式提交注册信息，post的数据都保存在initial_data里面
        # username就是用户注册的手机号，验证码按添加时间倒序排序，为了后面验证过期，错误等
        verify_records = VerifyCode.objects.filter(mobile=self.initial_data['username']).order_by("-add_time")

        if verify_records:
            # 最近的一个验证码
            last_record = verify_records[0]
            # 有效为5分钟
            five_minutes_ago = datetime.now() - timedelta(hours=0, minutes=5, seconds=0)
            if five_minutes_ago > last_record.add_time:
                raise serializers.ValidationError("验证码过期")

            if last_record.code != code:
                # 说明输入的验证码是错误的 和数据中匹配不上
                raise serializers.ValidationError("验证码错误")
        else:
            raise serializers.ValidationError("请确认账号")



    # 所有字段验证完后的方法 attrs是字段验证合法后返回的dict
    def validate(self, attrs):
        # 前端没有传mobile值到后端，这里添加进来
        attrs["mobile"] = attrs["username"]
        # code是自己添加得，数据库中并没有这个字段，验证完就删除掉
        del attrs["code"]
        return attrs

    class Meta:
        model = User
        fields = ('username', 'code', 'mobile', 'password')
```

序列化类中将code、username、password字段重新定义。

保存密码因为UserProfile应该是加密密码，如果用model自定义的create会保存明文，所以可以重载create方法来保存。或者使用下面的信号量的方法。



视图集

```python
class UserViewSet(CreateModelMixin, viewsets.GenericViewSet):
    """
    用户注册视图
    """
    serializer_class = UserRegSerializer
    queryset = User.objects.all()
```







## django信号量实现用户密码修改

Django 1.11 中文翻译文档 信号量部分：https://yiyibooks.cn/xx/Django_1.11.6/topics/signals.html

> users/signals.py

```python
#!/usr/bin/env python
# -*- coding:utf-8 -*-
#    @Author:iSk2y

from django.db.models.signals import post_save
from django.dispatch import receiver
from django.contrib.auth import get_user_model
User = get_user_model()


# post_save:接收信号的方式，在save后
# sender: 接收信号的model
@receiver(post_save, sender=User)
def create_user(sender, instance=None, created=False, **kwargs):
    # 是否新建，因为update的时候也会进行post_save
    if created:
        password = instance.password
        # instance相当于user
        instance.set_password(password)
        instance.save()
```

在user也就是UserProfile保存后会触发这个方法，发送给这个方法这个信号。然后就执行这个方法。不过先得让这个脚本引入到Django的环境变量中，就在apps.py中来import吧

```python
from django.apps import AppConfig


class UsersConfig(AppConfig):
    name = 'users'
    verbose_name = '用户管理'

    def ready(self):
        import users.signals
```



## 自定注册后jwt返回

现在可以成功可以注册用户了，那么注册完之后应该返回jwt的auth和前端对应，vue中的代码是这样写的

```javascript
isRegister(){
            var that = this;
            register({
                password:that.password,
                username:that.mobile ,
                code:that.code,
            }).then((response)=> {
              cookie.setCookie('name',response.data.username,7);
              cookie.setCookie('token',response.data.token,7)
              //存储在store
              // 更新store数据
              that.$store.dispatch('setInfo');
              //跳转到首页页面
              this.$router.push({ name: 'index'})

          })
          .catch(function (error) {
            that.error.mobile = error.username?error.username[0]:'';
            that.error.password = error.password?error.password[0]:'';
            that.error.username = error.mobile?error.mobile[0]:'';
            that.error.code = error.code?error.code[0]:'';
          });
        },
```

那在我们成功注册后，就要返回name和token了，

### 如何返回？

来看看jwt他源码中是如何返回的？

```python
class JSONWebTokenSerializer(Serializer):
    """
    Serializer class used to validate a username and password.

    'username' is identified by the custom UserModel.USERNAME_FIELD.

    Returns a JSON Web Token that can be used to authenticate later calls.
    """
    def __init__(self, *args, **kwargs):
        """
        Dynamically add the USERNAME_FIELD to self.fields.
        """
        super(JSONWebTokenSerializer, self).__init__(*args, **kwargs)

        self.fields[self.username_field] = serializers.CharField()
        self.fields['password'] = PasswordField(write_only=True)

    @property
    def username_field(self):
        return get_username_field()

    def validate(self, attrs):
        credentials = {
            self.username_field: attrs.get(self.username_field),
            'password': attrs.get('password')
        }

        if all(credentials.values()):
            user = authenticate(**credentials)

            if user:
                if not user.is_active:
                    msg = _('User account is disabled.')
                    raise serializers.ValidationError(msg)

                payload = jwt_payload_handler(user) # 重点在这里

                return {
                    'token': jwt_encode_handler(payload), # token是encode这样生成的
                    'user': user
                }
            else:
                msg = _('Unable to log in with provided credentials.')
                raise serializers.ValidationError(msg)
        else:
            msg = _('Must include "{username_field}" and "password".')
            msg = msg.format(username_field=self.username_field)
            raise serializers.ValidationError(msg)

```



那我们在viewset用户视图集中重载下create的方法，来增加这个返回内容就好了

```python
from rest_framework_jwt.serializers import jwt_encode_handler, jwt_payload_handler

class UserViewSet(CreateModelMixin, viewsets.GenericViewSet):
    """
    用户注册视图
    """
    serializer_class = UserRegSerializer
    queryset = User.objects.all()

    def create(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        # 把序列化后的数据赋值给user
        user = self.perform_create(serializer)
        re_dict = serializer.data
        # 实例化payload
        payload = jwt_payload_handler(user)
        re_dict['token'] = jwt_encode_handler(payload)
        re_dict['name'] = user.name if user.name else user.username

        headers = self.get_success_headers(serializer.data)
        return Response(re_dict, status=status.HTTP_201_CREATED, headers=headers)

    def perform_create(self, serializer):
        return serializer.save()
```

重载perform_create是为了return回来user这个实例。

![drf测试](https://upload-images.jianshu.io/upload_images/14657587-bf52f2e39488c039.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


注册功能暂时完成。云片网发短信未实际测试。





# 目前完成图



![目前效果](https://upload-images.jianshu.io/upload_images/14657587-257e8b29b84964de.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


