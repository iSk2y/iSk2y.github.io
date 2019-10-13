---
title: Vue+drf前后端分离之生鲜电商项目(二)
date: 2018-12-01 16:14:45
tags:
  - Django
  - Python
  - Vue
categories: Django
---

接着 [Vue+drf前后端分离之生鲜电商项目(一)](https://isk2y.github.io/2018/11/28/vue-drf-onlineshop/) 继续下面的api开发笔记记录

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



# 商品详情页

```python
class GoodsListViewSet(mixins.ListModelMixin, mixins.RetrieveModelMixin, viewsets.GenericViewSet):
    """
    list:
        商品列表数据
    """
    queryset = Goods.objects.all().order_by("id")
    pagination_class = GoodsPagination
    filter_backends = (DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter)
    filter_class = GoodsFilter
    # 设置filter的类为我们自定义的类
    serializer_class = GoodSerializer

    # 搜索
    search_fields = ('name', 'goods_brief', 'goods_desc')
    # 排序
    ordering_fields = ('sold_num', 'shop_price')
```

之前GoodsListViewSet视图集类，得益于drf的router，所以再多继承一个mixins.RetrieveModelMixin类就能查看详细了

```python
# Lib/site-packages/rest_framework/routers.py
routes = [
        # List route.
        Route(
            url=r'^{prefix}{trailing_slash}$',
            mapping={
                'get': 'list',
                'post': 'create'
            },
            name='{basename}-list',
            detail=False,
            initkwargs={'suffix': 'List'}
        ),
        # Dynamically generated list routes. Generated using
        # @action(detail=False) decorator on methods of the viewset.
        DynamicRoute(
            url=r'^{prefix}/{url_path}{trailing_slash}$',
            name='{basename}-{url_name}',
            detail=False,
            initkwargs={}
        ),
        # Detail route.
        Route(
            url=r'^{prefix}/{lookup}{trailing_slash}$',
            mapping={
                'get': 'retrieve',
                'put': 'update',
                'patch': 'partial_update',
                'delete': 'destroy'
            },
            name='{basename}-detail',
            detail=True,
            initkwargs={'suffix': 'Instance'}
        ),
        # Dynamically generated detail routes. Generated using
        # @action(detail=True) decorator on methods of the viewset.
        DynamicRoute(
            url=r'^{prefix}/{lookup}/{url_path}{trailing_slash}$',
            name='{basename}-{url_name}',
            detail=True,
            initkwargs={}
        ),
    ]
```



GoodsImage是Goods的图片，其中goods字段外键关联了Goods，所以在序列化类中需要定义下覆盖

```python
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
```

```python
class GoodsImageSerializer(serializers.ModelSerializer):
    class Meta:
        model = GoodsImage
        fields = ("image",)
    
    
class GoodSerializer(serializers.ModelSerializer):
    """
    商品序列化
    """
    category = CategorySerializer()
    # images是数据库中设置的related_name="images"
    images = GoodsImageSerializer(many=True)
    
    class Meta:
        model = Goods
        fields = '__all__'
```



# 热卖接口

Vue中热卖商品是这样请求的

```javascript
getHotSales() { //请求热卖商品
              getGoods({
                is_hot:true
              })
                .then((response)=> {
                    console.log(response.data)
                    this.hotProduct = response.data.results;

                }).catch(function (error) {
                    console.log(error);
                });
            }
```

而getGoods用的接口就是

```javascript
//获取商品列表
export const getGoods = params => { return axios.get(`${local_host}/goods/`, { params: params }) }
```



在过滤器中增加“is_hot”就可以了

> goods/filters.py

```python
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
        fields = ['pricemin', 'pricemax','is_hot'] # 加is_hot
```

在后台设置商品的“is_hot”为True,然后前端就可以显示出来了



# 用户收藏

## 序列化类

```python
#!/usr/bin/env python
# -*- coding:utf-8 -*-
#    @Author:iSk2y

from rest_framework import serializers
from user_operation.models import UserFav
from rest_framework.validators import UniqueTogetherValidator


class UserFavSerializer(serializers.ModelSerializer):
    # 获取当前登录的用户
    user = serializers.HiddenField(
        default=serializers.CurrentUserDefault()
    )

    class Meta:
        # validate实现唯一联合，一个商品只能收藏一次
        validators = [
            UniqueTogetherValidator(
                queryset=UserFav.objects.all(),
                fields=('user', 'goods'),
                # message的信息可以自定义
                message="已经收藏"
            )
        ]
        model = UserFav
        # 收藏的时候需要返回商品的id，因为取消收藏的时候必须知道商品的id是多少
        fields = ("user", "goods",'id')
```

CurrentUserDefault获取到当前登录的用户（从request中）

UniqueTogetherValidator 联合唯一



## 视图集

```python
from rest_framework import viewsets
from rest_framework import mixins
from .models import UserFav
from .serializers import UserFavSerializer


class UserFavViewset(viewsets.GenericViewSet, mixins.ListModelMixin, mixins.CreateModelMixin, mixins.DestroyModelMixin):
    """
    用户收藏视图集
    """
    queryset = UserFav.objects.all()
    serializer_class = UserFavSerializer
    pagination_class = None
```

ListModelMixin 获取收藏列表

CreateModelMixin 创建收藏操作，在序列化中做了联合唯一，不会重复收藏

DestroyModelMixin 删除收藏操作

## url

```python
from user_operation import views as uoview
# 配置用户收藏的url
router.register(r'userfavs', uoview.UserFavViewset, base_name="userfavs")
```



## 操作权限认证

drf中文翻译文档权限部分：http://drf.jiuyou.info/#/drf/permissions?id=%E5%A6%82%E4%BD%95%E7%A1%AE%E5%AE%9A%E6%9D%83%E9%99%90



逻辑上用户不能浏览和删除别人的收藏记录，所以采用权限认证类

```python
# utils/permissions.py
#!/usr/bin/env python
# -*- coding:utf-8 -*-
#    @Author:iSk2y

from rest_framework import permissions


class IsOwnerOrReadOnly(permissions.BasePermission):
    """
    Object-level permission to only allow owners of an object to edit it.
    Assumes the model instance has an `owner` attribute.
    """

    def has_object_permission(self, request, view, obj):
        # Read permissions are allowed to any request,
        # so we'll always allow GET, HEAD or OPTIONS requests.
        if request.method in permissions.SAFE_METHODS:
            return True

        # Instance must have an attribute named `owner`.
        return obj.user == request.user
```

视图集

```python
# user_operation/views.py
from rest_framework import viewsets
from rest_framework import mixins
from .models import UserFav
from .serializers import UserFavSerializer
from rest_framework_jwt.authentication import JSONWebTokenAuthentication
from rest_framework.authentication import SessionAuthentication
from rest_framework.permissions import IsAuthenticated
from utils.permissions import IsOwnerOrReadOnly


class UserFavViewset(viewsets.GenericViewSet, mixins.ListModelMixin, mixins.CreateModelMixin, mixins.DestroyModelMixin):
    """
    用户收藏视图集
    """
    lookup_field = 'goods_id'
    serializer_class = UserFavSerializer
    pagination_class = None
    # 认证类
    authentication_classes = (JSONWebTokenAuthentication, SessionAuthentication)
    # 权限认证类
    # IsAuthenticated：必须登录用户；IsOwnerOrReadOnly：必须是当前登录的用户
    permission_classes = (IsAuthenticated, IsOwnerOrReadOnly)
    
    def get_queryset(self):
        # 返回当前用户的列表
        return UserFav.objects.filter(user=self.request.user)
```

主要逻辑：

1. 用户只能获取到自己的收藏列表 - > 重载了get_queryset
2. 只有登录用户才能够收藏 - > permission_classes
3. 只有用户自己才能够删除或者添加自己的收藏记录 - > IsOwnerOrReadOnly



Vue中api

```javascript
//收藏
export const addFav = params => { return axios.post(`${host}/userfavs/`, params) }

//取消收藏
export const delFav = goodsId => { return axios.delete(`${host}/userfavs/`+goodsId+'/') }

export const getAllFavs = () => { return axios.get(`${host}/userfavs/`) }

//判断是否收藏
export const getFav = goodsId => { return axios.get(`${host}/userfavs/`+goodsId+'/') }

```





没有登录，无法获取（get）

![favget.jpg](https://upload-images.jianshu.io/upload_images/14657587-471f4b8cb43998c5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



jwt验证后加入Header中，再get可以获取，而且是当前自己的列表

![image.png](https://upload-images.jianshu.io/upload_images/14657587-45e208772bbcd86a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



delete方法 删除收藏记录，但是必须先通过jwt验证

![image.png](https://upload-images.jianshu.io/upload_images/14657587-415e4808f6986021.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



# api文档自动生成

```

    path('docs',include_docs_urls(title='慕学生鲜')),
```

![image.png](https://upload-images.jianshu.io/upload_images/14657587-55f3cfd82091de21.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

只能说tql

# 用户个人中心

之前写过用户的一个视图集，这里可以复用，再继承一个mixins.RetrieveModelMixin

但是之前视图集中指定了序列化类，所以这里要重载下get_serializer_class让他动态返回序列化类，序列化我们想要的内容

先来写我们的详细信息序列化类

```python
class UserDetailSerializer(serializers.ModelSerializer):
    """
    用户详情序列化类
    """
    class Meta:
        model = User
        fields = ("name", "gender", "birthday", "email", "mobile")
```

重载get_serializer_class

```python
    def get_serializer_class(self):
        """
        动态选择序列化类，区分注册和详细
        :return:
        """
        if self.action == 'retrieve':
            return UserDetailSerializer
        elif self.action == 'create':
            return UserRegSerializer
        
        return UserDetailSerializer
```

同理权限认证类也要重载（查看个人详情要授权，注册的时候总不能让他先授权，那还注册啥哈？）

```python
   def get_permissions(self):
        """
        动态选择权限认证类
        :return:
        """
        if self.action == 'retrieve' or self.action == 'update':
            return [IsAuthenticated()]
        else:
            return []

    # 虽然继承了Retrieve可以获取用户详情，但是并不知道用户的id，所有要重写get_object方法
    # 重写get_object方法，就知道是哪个用户了
    def get_object(self):
        return self.request.user
```

修改信息就再继承一个 UpdateModelMixin



```python
class UserViewSet(CreateModelMixin, RetrieveModelMixin, UpdateModelMixin, viewsets.GenericViewSet):
    """
    create:
        用户注册
    retrieve:
        用户个人详细信息
    update:
        修改用户个人信息
    """
    serializer_class = UserRegSerializer
    queryset = User.objects.all()
    authentication_classes = (JSONWebTokenAuthentication, SessionAuthentication)

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

    def get_serializer_class(self):
        """
        动态选择序列化类，区分注册和详细
        :return:
        """
        if self.action == 'retrieve':
            return UserDetailSerializer
        elif self.action == 'create':
            return UserRegSerializer

        return UserDetailSerializer

    def get_permissions(self):
        """
        动态选择权限认证类
        :return:
        """
        if self.action == 'retrieve' or self.action == 'update':
            return [IsAuthenticated()]
        else:
            return []

    # 虽然继承了Retrieve可以获取用户详情，但是并不知道用户的id，所有要重写get_object方法
    # 重写get_object方法，就知道是哪个用户了
    def get_object(self):
        return self.request.user
```



# 用户收藏记录

用户个人中心收藏记录得看到详细内容，重新写个序列化类，并将good覆写序列化成商品信息

```python
class UserFavDetailSerializer(serializers.ModelSerializer):
    """
    用户收藏详情
    """

    # 通过商品id获取收藏的商品，需要嵌套商品的序列化
    goods = GoodSerializer()

    class Meta:
        model = UserFav
        fields = ("goods", "id")
```

也需要动态获取序列化类，就重写get_serializer_class

```python
    def get_serializer_class(self):
        # 动态选择序列化类
        if self.action == 'list':
            return UserFavDetailSerializer
        elif self.action == 'create':
            return UserFavSerializer
        return UserFavSerializer
```



# 用户留言

序列化类

```python
class LeavingMessageSerializer(serializers.ModelSerializer):
    """
    用户留言
    """
    user = serializers.HiddenField(
        default=serializers.CurrentUserDefault()
    )

    add_time = serializers.DateTimeField(read_only=True, format='%Y-%m-%d %H:%M')
    
    class Meta:
        model = UserLeavingMessage
        fields = ("user", "message_type", "subject", "message", "file", "id", "add_time")
```

视图集

```python
class LeavingMessageViewset(mixins.ListModelMixin, mixins.DestroyModelMixin, mixins.CreateModelMixin,
                            viewsets.GenericViewSet):
    """
    list:
        获取用户留言
    create:
        添加留言
    delete:
        删除留言功能
    """

    permission_classes = (IsAuthenticated, IsOwnerOrReadOnly)
    authentication_classes = (JSONWebTokenAuthentication, SessionAuthentication)
    serializer_class = LeavingMessageSerializer

    # 只能看到自己的留言
    def get_queryset(self):
        return UserLeavingMessage.objects.filter(user=self.request.user)
```

Vue中的api

```javascript
//获取留言
export const getMessages = () => {return axios.get(`${host}/messages/`)}

//添加留言
export const addMessage = params => {return axios.post(`${host}/messages/`, params, {headers:{ 'Content-Type': 'multipart/form-data' }})}

//删除留言
export const delMessages = messageId => {return axios.delete(`${host}/messages/`+messageId+'/')}

```





# 用户收货地址

序列化类

```python
class AddressSerializer(serializers.ModelSerializer):
    user = serializers.HiddenField(
        default=serializers.CurrentUserDefault()
    )
    add_time = serializers.DateTimeField(read_only=True, format='%Y-%m-%d %H:%M')

    class Meta:
        model = UserAddress
        fields = ("id", "user", "province", "city", "district", "address", "signer_name", "add_time", "signer_mobile")
```



视图集

```python
class AddressViewset(viewsets.ModelViewSet):
    """
    收货地址管理
    list:
        获取收货地址
    create:
        添加收货地址
    update:
        更新收货地址
    delete:
        删除收货地址
    """
    permission_classes = (IsAuthenticated, IsOwnerOrReadOnly)
    authentication_classes = (JSONWebTokenAuthentication, SessionAuthentication)
    serializer_class = AddressSerializer

    def get_queryset(self):
        return UserAddress.objects.filter(user=self.request.user)
```



配置下url

```python
# 配置收货地址
router.register(r'address',AddressViewset , base_name="address")
```

Vue中api接口

```javascript
//添加收货地址
export const addAddress = params => {return axios.post(`${host}/address/`, params)}

//删除收货地址
export const delAddress = addressId => {return axios.delete(`${host}/address/`+addressId+'/')}

//修改收货地址
export const updateAddress = (addressId, params) => {return axios.patch(`${host}/address/`+addressId+'/', params)}

//获取收货地址
export const getAddress = () => {return axios.get(`${host}/address/`)}

```



# 购物车

Vue中的api接口url

```javascript
//获取购物车商品
export const getShopCarts = params => { return axios.get(`${host}/shopcarts/`) }
// 添加商品到购物车
export const addShopCart = params => { return axios.post(`${host}/shopcarts/`, params) }
//更新购物车商品信息
export const updateShopCart = (goodsId, params) => { return axios.patch(`${host}/shopcarts/`+goodsId+'/', params) }
//删除某个商品的购物记录
export const deleteShopCart = goodsId => { return axios.delete(`${host}/shopcarts/`+goodsId+'/') }

```

序列化类

```python
from .models import ShoppingCart
from rest_framework import serializers
from goods.models import Goods
from goods.serializers import GoodSerializer


class ShopCartDetailSerializer(serializers.ModelSerializer):
    """
    购物车商品详情信息
    """
    goods = GoodSerializer(many=False, read_only=True)

    class Meta:
        model = ShoppingCart
        fields = ("goods", "nums")


class ShopCartSerializer(serializers.Serializer):

    # 获取当前登录的用户
    user = serializers.HiddenField(
        default=serializers.CurrentUserDefault()
    )
    nums = serializers.IntegerField(required=True, label="数量", min_value=1,
                                    error_messages={
                                        "min_value": "商品数量不能小于一",
                                        "required": "请选择购买数量"
                                    })

    # goods是一个外键，可以通过这方法获取goods object中所有的值
    goods = serializers.PrimaryKeyRelatedField(required=True, queryset=Goods.objects.all())

    def create(self, validated_data):
        # validated_data是已经处理过的数据
        # 获取当前用户
        # view中:self.request.user；serizlizer中:self.context["request"].user
        user = self.context["request"].user
        nums = validated_data["nums"]
        goods = validated_data["goods"]

        existed = ShoppingCart.objects.filter(user=user, goods=goods)

        if existed:
            existed = existed[0]
            existed.nums += nums
            existed.save()
        else:
            # 添加到购物车
            existed = ShoppingCart.objects.create(**validated_data)

        return existed

    def update(self, instance, validated_data):
        # 修改商品数量
        instance.nums = validated_data["nums"]
        instance.save()
        return instance
```

视图集

```python
class AddressViewset(viewsets.ModelViewSet):
    """
    收货地址管理
    list:
        获取收货地址
    create:
        添加收货地址
    update:
        更新收货地址
    delete:
        删除收货地址
    """
    permission_classes = (IsAuthenticated, IsOwnerOrReadOnly)
    authentication_classes = (JSONWebTokenAuthentication, SessionAuthentication)
    serializer_class = AddressSerializer
    pagination_class = None

    def get_queryset(self):
        return UserAddress.objects.filter(user=self.request.user)
```



# 订单

1. 结算购物车中的goods
2. 选择地址
3. 结算生成订单
4. 删除购物车中的对应商品
5. 在个人中心看到订单

序列化类

```python
class OrderGoodsSerializer(serializers.ModelSerializer):
    """
    订单中的商品序列化类
    """
    goods = GoodSerializer(many=False)

    class Meta:
        model = OrderGoods
        fields = "__all__"


class OrderDetailSerializer(serializers.ModelSerializer):
    """
    订单详细信息
    """
    goods = OrderGoodsSerializer(many=True)

    class Meta:
        model = OrderInfo
        fields = "__all__"


class OrderSerializer(serializers.ModelSerializer):
    """
    订单序列化
    """
    user = serializers.HiddenField(
        default=serializers.CurrentUserDefault()
    )
    # 生成订单时这些不用post
    pay_status = serializers.CharField(read_only=True)
    trade_no = serializers.CharField(read_only=True)
    order_sn = serializers.CharField(read_only=True)
    pay_time = serializers.DateTimeField(read_only=True)
    nonce_str = serializers.CharField(read_only=True)
    pay_type = serializers.CharField(read_only=True)

    def generate_order_sn(self):
        # 生成订单号
        # 当前时间+userid+随机数
        from random import Random
        import time
        random_ins = Random()
        order_sn = "{time_str}{userid}{ranstr}".format(time_str=time.strftime("%Y%m%d%H%M%S"),
                                                       userid=self.context["request"].user.id,
                                                       ranstr=random_ins.randint(10, 99))
        return order_sn

    def validate(self, attrs):
        # validate中添加order_sn，然后在view中就可以save
        attrs["order_sn"] = self.generate_order_sn()
        return attrs

    class Meta:
        model = OrderInfo
        fields = "__all__"
```

视图集

```python
class OrderViewset(mixins.ListModelMixin, mixins.RetrieveModelMixin, mixins.CreateModelMixin, mixins.DestroyModelMixin,
                   viewsets.GenericViewSet):
    """
    订单管理
    list:
        获取个人订单
    delete:
        删除订单
    create：
        新增订单
    """
    permission_classes = (IsAuthenticated, IsOwnerOrReadOnly)
    authentication_classes = (JSONWebTokenAuthentication, SessionAuthentication)
    serializer_class = OrderSerializer
    pagination_class = None

    # 动态配置serializer
    def get_serializer_class(self):
        if self.action == "retrieve":
            return OrderDetailSerializer
        return OrderSerializer

    # 获取订单列表
    def get_queryset(self):
        return OrderInfo.objects.filter(user=self.request.user)

    # 在订单提交保存之前还需要多两步步骤，所以这里自定义perform_create方法
    # 1.将购物车中的商品保存到OrderGoods中
    # 2.清空购物车
    def perform_create(self, serializer):
        order = serializer.save()
        # 获取购物车所有商品
        shop_carts = ShoppingCart.objects.filter(user=self.request.user)
        for shop_cart in shop_carts:
            order_goods = OrderGoods()
            order_goods.goods = shop_cart.goods
            order_goods.goods_num = shop_cart.nums
            order_goods.order = order
            order_goods.save()
            # 清空购物车
            shop_cart.delete()
        return order

```



# 支付功能

支付宝支付接口开发，因个别原因，暂时不做模拟测试。





# 其他细节接口

## 轮播

```python
# goods/serializers.py
class BannerSerializer(serializers.ModelSerializer):
    """
    轮播图
    """
    class Meta:
        model = Banner
        fields = "__all__"
```



```python
# goods/views.py
class BannerViewset(mixins.ListModelMixin, viewsets.GenericViewSet):
    """
    首页轮播图
    """
    queryset = Banner.objects.all().order_by("index")
    serializer_class = BannerSerializer
```



## 新品接口

在设计Goods model时候有一个字段is_new

```
is_new = models.BooleanField("是否新品",default=False)
```

实现这个接口只要在goods/filters/GoodsFilter里面添加一个过滤就可以了

```
    class Meta:
        model = Goods
        fields = ['pricemin', 'pricemax','is_hot','is_new']
```



## 信号量处理收藏数

```python
# user_operation/signals.py
#!/usr/bin/env python
# -*- coding:utf-8 -*-
#    @Author:iSk2y

#
# 要在apps.py中引入下 我没引入 注释了 感觉在serializer中重载更灵活
#
#
#

from django.db.models.signals import post_save,post_delete
from django.dispatch import receiver
from user_operation.models import UserFav

# post_save:接收信号的方式
# sender: 接收信号的model


@receiver(post_save, sender=UserFav)
def create_UserFav(sender, instance=None, created=False, **kwargs):
    # 是否新建，因为update的时候也会进行post_save
    if created:
        goods = instance.goods
        goods.fav_num += 1
        goods.save()


@receiver(post_delete, sender=UserFav)
def delete_UserFav(sender, instance=None, created=False, **kwargs):
        goods = instance.goods
        goods.fav_num -= 1
        goods.save()
```



```python
# user_operation/apps.py
from django.apps import AppConfig


class UserOperationConfig(AppConfig):
    name = 'user_operation'
    verbose_name = '操作管理'

    def ready(self):
        import user_operation.signals
```



## 库存数

我觉得库存数变化应该在订单生成后，而不是在加入购物车后吧？

视频教程中还是放在了加入购物车后

```python
def perform_create(self, serializer):
        shop_cart = serializer.save()
        goods = shop_cart.goods
        goods.goods_num -= shop_cart.nums
        goods.save()

    # 库存数+1
    def perform_destroy(self, instance):
        goods = instance.goods
        goods.goods_num += instance.nums
        goods.save()
        instance.delete()

    # 更新库存,修改可能是增加页可能是减少
    def perform_update(self, serializer):
        #首先获取修改之前的库存数量
        existed_record = ShoppingCart.objects.get(id=serializer.instance.id)
        existed_nums = existed_record.nums
        # 先保存之前的数据existed_nums
        saved_record = serializer.save()
        #变化的数量
        nums = saved_record.nums-existed_nums
        goods = saved_record.goods
        goods.goods_num -= nums
        goods.save()
```



## 商品销量

因没有做支付宝接口，回调也没写，所以这里暂时不写了



# drf缓存设置

drf的一个扩展来实现缓存，github上面的使用说明：http://chibisov.github.io/drf-extensions/docs/#caching

![image.png](https://upload-images.jianshu.io/upload_images/14657587-6f03fa9996f4073e.png)

1. 安装

```
pip install drf-extensions
```

2. 简单使用

这里最简单的使用继承mixin

```python
from rest_framework_extensions.cache.mixins import CacheResponseMixin

class GoodsListViewSet(CacheResponseMixin, mixins.ListModelMixin, mixins.RetrieveModelMixin, viewsets.GenericViewSet):
    """
    list:
        商品列表数据
    """
```

把缓存mixin继承在第一个。

这里默认缓存在了内存中，如果重新启动就会丢失，要重新缓存。当然缓存还有很多其他配置，比如时间等等

```python
REST_FRAMEWORK_EXTENSIONS = {
    'DEFAULT_CACHE_RESPONSE_TIMEOUT': 5   #5s过期，时间自己可以随便设定
}
```





![缓存第一次访问](https://upload-images.jianshu.io/upload_images/14657587-d771927d768bc8b1.png)

![再次访问](https://upload-images.jianshu.io/upload_images/14657587-07db78b789be2d47.png)

缓存第一次和后次访问速度



# drf的throttle访问速率

```python
REST_FRAMEWORK = {
    # 限速设置
    'DEFAULT_THROTTLE_CLASSES': (
            'rest_framework.throttling.AnonRateThrottle',   # 未登陆用户
            'rest_framework.throttling.UserRateThrottle'    # 登陆用户
        ),
    'DEFAULT_THROTTLE_RATES': {
        'anon': '3/minute',         # 每分钟可以请求两次
        'user': '5/minute'          # 每分钟可以请求五次
    }
}
```

```python
from rest_framework.throttling import UserRateThrottle,AnonRateThrottle

class GoodsListViewSet(CacheResponseMixin,mixins.ListModelMixin, mixins.RetrieveModelMixin,viewsets.GenericViewSet):
　　.
　　.
　　throttle_classes = (UserRateThrottle, AnonRateThrottle)
```



第三方登陆和sentry不做记录了



# 告诉自己

发现跟学了两个实际项目下来，代码是别人的终归是别人的，不是完全自主开发是没有很多感受的。

告诉自己：切勿眼高手低。要专注。不要浮躁，加油！！努力不一定可以成功，但不努力肯定失败！
