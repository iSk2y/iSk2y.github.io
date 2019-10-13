---
title: Django实战MxOnline之模型设计
date: 2018.11.04 22:34
tags:
  - Django
  - Python
categories: Django
---

学习Django有段时间了，找个例子来跟学下项目流程和经验

遂找到了 Django+xadmin打造在线教育平台 这个专题，也找到了 两个记录该专题学习过程中的笔记 

<!-- more -->

[Zhang_derek](http://www.cnblogs.com/derek1184405959/p/8590360.html) 

[mtianyan](https://www.jianshu.com/nb/21010157) 

在阅读的基础上，记录下自己遇到的一些问题和笔记吧！

# 开发环境

- Python 3.6.6
- Django 2.0
- OS：Windows （其实我是很想在Linux上开发的，奈何我开个虚拟机就炸锅了）
- MySQL：5.7



## 虚拟环境

- virtualenv
- virtualenvwrapper：到Python的scripts文件夹下修改mkvirtualenv.bat文件的24行，可以修改创建env的根目录



## mysql驱动库

视频中使用的是python-mysql，后改mysql-client

如果使用pymysql库

在项目`__init__.py`中加入

```python
import pymysql
pymysql.install_as_MySQLdb()
```

顺便还去查了下，貌似mysqlclient库比pymysql性能好很多。于是下载了[mysqlclient](https://github.com/PyMySQL/mysqlclient-python) 注意对应版本。然后直接pip文件安装就行。



# 应用和模型设计

解决循环引用，分层设计。

![分层设计](http://upload-images.jianshu.io/upload_images/14657587-59bb4427a8fb55f1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## users-models

- userprofile：用户信息表。扩展user表，添加其他字段。其中有个[imageField](https://docs.djangoproject.com/zh-hans/2.0/ref/models/fields/#imagefield)字段以前没用到过，Mark
- EmailVerifyRecord：邮箱验证码表
- banner：轮播图的表



## courses-models

- Course：课程基本信息
- Lesson：章节信息
- Video：视频
- CourseResource：课程资源

课程 - 章节 = 》一对多。

章节 - 视频 =》一对多。

课程 - 资源 =》一对多。



## organization-models

- CityDict：城市表。（但是我感觉，好像不是太有必要。。）
- CourseOrg：课程机构表
- Teacher：教师表



## operation-models

- UserAsk：用户咨询表，用来课前咨询的，注意这里的Course没有用外键哦

- CourseComments：课程评论表

- UserFavorite：用户收藏表。这里的收藏使用了 id和type两个字段来定位特定对某个类型的收藏，而不是采用外键

  ```python
  class UserFavorite(models.Model):
      """
      用户收藏动作表
      """
      FAV_CHOICES = (  # 收藏类型
          (1, '课程'),
          (2, '课程机构'),
          (3, '讲师')
      )
      user = models.ForeignKey(to=UserProfile, verbose_name='用户', on_delete=models.CASCADE)
      fav_id = models.IntegerField('数据id', default=0)
      fav_type = models.IntegerField(verbose_name='收藏类型', choices=FAV_CHOICES, default=1)
      # 这里用fav_id来存放对应收藏类型的对象id，而不采取直接用外键，这样减少存储空间而且更灵活
      add_time = models.DateTimeField(verbose_name='添加时间', default=datetime.now)
  
      class Meta:
          verbose_name = '用户收藏'
          verbose_name_plural = verbose_name
  ```

- UserMessage：用户消息表。这里用户的字段也没有用外键。而是用了int，这样可以更好实现全员消息

  ```python
  class UserMessage(models.Model):
      """
      用户消息表
      """
      # 这边user不使用外键，使得我们可以更好实现全员消息和特定用户消息的推送 ，默认0就是全员消息
      user = models.IntegerField(default=0, verbose_name='用户')
      message = models.CharField(verbose_name='消息内容', max_length=500)
      has_read = models.BooleanField(verbose_name='是否已读', default=False)
      send_time = models.DateTimeField(verbose_name='发送时间', default=datetime.now)
  
      class Meta:
          verbose_name = "用户消息"
          verbose_name_plural = verbose_name
  ```

- UserCourse：用户课程表。



UserMessage和UserFavorite这两个model的定义方式 我应该认真思考好好吸收！Mark

> PS：包导入问题，可以右键文件夹（包）设为SourceRoot，这样可以设为根目录

# 安装xadmin

[Xadmin for Django2.0](https://github.com/sshwsfc/xadmin/tree/django2) 

```shell
pip install https://codeload.github.com/sshwsfc/xadmin/zip/django2
```

速度可能有点慢，可以手动下载那个zip，然后pip install

- 安装完，记得注册进APP然后修改URL路由，再migrate一下生成表
- 记得在APP中新建adminx.py，这样xadmin才会自动发现到。



配置类的定义：不同于admin，这里继承自object，主要配置选项

- list_display：定义显示字段  - 应该是配置select的字段
- search_fields：定义搜索字段 - 应该是利用like
- list_filter：定义过滤字段 - 应该是 group by



## BaseAdminView

配置一些选项和设置。

### 主题

```python
#  Xadmin管理配置类
class BaseSetting(object):
    # 开启主题
    enable_themes = True
    use_bootswatch = True
    
xadmin.site.register(views.BaseAdminView, BaseSetting)
```

PS：Journal橙红色不错、Flatly绿和SANDSTIONE浅绿、Lumen白色简洁（荐）、simplex红、spacelab蓝和yeti浅蓝、



## CommAdminView

```python
# x admin 全局配置参数信息设置
class GlobalSettings(object):
    site_title = "MxOnline"
    site_footer = "MxOnline Platform"
    # 收起菜单
    menu_style = "accordion"
# 将头部与脚部信息进行注册:
xadmin.site.register(views.CommAdminView, GlobalSettings)
```

APP名称默认显示为英文的原名，可以进行修改自定义显示。

1. 在app的目录下apps.py对应Config类中设置 verbose_name属性
2. 在app的目录下`__init__.py`中将app配置类指定

```python
default_app_config = "operation.apps.OperationConfig" # 对应修改
```



### 自定义菜单顺序

```python
def get_site_menu(self):
        return (
            {'title': '课程管理', 'menus': (
                {'title': '课程信息', 'url': self.get_model_url(Course, 'changelist')},
                {'title': '章节信息', 'url': self.get_model_url(Lesson, 'changelist')},
                {'title': '视频信息', 'url': self.get_model_url(Video, 'changelist')},
                {'title': '课程资源', 'url': self.get_model_url(CourseResource, 'changelist')},
                {'title': '课程评论', 'url': self.get_model_url(CourseComments, 'changelist')},
            )},
            {'title': '机构管理', 'menus': (
                {'title': '所在城市', 'url': self.get_model_url(CityDict, 'changelist')},
                {'title': '机构讲师', 'url': self.get_model_url(Teacher, 'changelist')},
                {'title': '机构信息', 'url': self.get_model_url(CourseOrg, 'changelist')},
            )},
            {'title': '用户管理', 'menus': (
                {'title': '用户信息', 'url': self.get_model_url(UserProfile, 'changelist')},
                {'title': '用户验证', 'url': self.get_model_url(EmailVerifyRecord, 'changelist')},
                {'title': '用户课程', 'url': self.get_model_url(UserCourse, 'changelist')},
                {'title': '用户收藏', 'url': self.get_model_url(UserFavorite, 'changelist')},
                {'title': '用户消息', 'url': self.get_model_url(UserMessage, 'changelist')},
            )},


            {'title': '系统管理', 'menus': (
                {'title': '用户咨询', 'url': self.get_model_url(UserAsk, 'changelist')},
                {'title': '首页轮播', 'url': self.get_model_url(Banner, 'changelist')},
                {'title': '用户分组', 'url': self.get_model_url(Group, 'changelist')},
                {'title': '用户权限', 'url': self.get_model_url(Permission, 'changelist')},
                {'title': '日志记录', 'url': self.get_model_url(Log, 'changelist')},
            )},
            )
```

记得要引入下每个models



基础模型设计完毕，后面可能会微调吧 到时候再修改。接下来开始 功能设计和开发。