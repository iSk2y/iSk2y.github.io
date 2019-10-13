---
title: Django实战MxOnline之功能设计
date: 2018.11.09 10:31:33
tags:
  - Django
  - Python
categories: Django
---

接着上一篇 设[计完基础模型和xadmin后台配置后](https://www.jianshu.com/p/2178cd37f258)，开始步入功能设计开发

给各位阅读的小伙伴提个醒，我这里只是简单记录下我认为之前没有生疏的，没有接触过的只是，如果要看详细解释可以看第一篇中我给出的链接blog

我对应的代码也在[GitHub](https://github.com/iSk2y/LearnMxOnline)上，欢迎下载！里面重要功能完成都commit了

<!-- more -->
# 登陆功能

## 自定义auth认证

[官方文档 自定义认证](https://docs.djangoproject.com/zh-hans/2.0/topics/auth/customizing/)

- 先在settings.py中设置auth允许的函数

```python
AUTHENTICATION_BACKENDS = (
    'users.views.CustomBackend',
)
```

- 然后在views中重载authenticate方法

```python
from django.contrib.auth.backends import ModelBackend
from .models import UserProfile
from django.db.models import Q

class CustomBackend(ModelBackend):
    # 继承ModelBackend 重载authenticate方法，使得邮件账户也能登陆
    def authenticate(self, request, username=None, password=None, **kwargs):
        try:
            # 返回0个和2个  get都会失败
            user = UserProfile.objects.get(Q(username=username)|Q(email=username))
            if user.check_password(password):  # 这里如果密码错误 user为None
                return user
        except Exception as e:
            return None
```



## FBV改成CBV

CBV比FBV会有很多好处，所以改写为CBV方式书写。（菜鸡我一直写FBV……）

- 在views中

```python
from django.views.generic.base import View
```

在views中定义的class要继承自这个基类View，然后书写get和post方法，记得传入request



- 在urls中

```python
path('login/', LoginView.as_view(), name='login')
```

要调用as_view()方法，这样会返回使用LoginView处理的函数句柄



# 注册功能

- 规范养成：在模板中使用static 便于后期维护，记得先load staticfiles

## captcha库

使用[django-simple-captcha](https://github.com/mbi/django-simple-captcha)这个库来快速实现验证码功能。python3直接`pip install  django-simple-captcha`

[官方使用手册](https://django-simple-captcha.readthedocs.io/en/latest/usage.html) ，这里简要记录

1. 要把captcha注册进`INSTALLED_APPS`
2. 然后添加`path('captcha', include('captcha.urls'))`
3. 在生成数据库 migrate一下。

这里注意下，点击图片刷新验证码是用前端js写的，手册上都有实例代码！



## 发送邮件

之前没有使用过Django发送过邮件，这里可以实验学习下，get！

*发送电子邮件的最简单方法是使用 django.core.mail.send_mail()。*

*的subject，message，from_email和recipient_list参数是必需的。*

- *subject：一个字符串。*
- *message：一个字符串。*
- *from_email：一个字符串。*
- *recipient_list：字符串列表，每个字符串都是电子邮件地址。每个成员都recipient_list将在电子邮件的“收件人：”字段中看到其他收件人。*
- *fail_silently：一个布尔值。如果是的话False，send_mail会提出一个smtplib.SMTPException。有关smtplib可能的例外列表，请参阅文档，所有这些例外都是。的子类 SMTPException。*
- *auth_user：用于向SMTP服务器进行身份验证的可选用户名。如果没有提供，Django将使用该EMAIL_HOST_USER设置的值 。*
- *auth_password：用于验证SMTP服务器的可选密码。如果没有提供，Django将使用该EMAIL_HOST_PASSWORD设置的值 。*
- *connection：用于发送邮件的可选电子邮件后端。如果未指定，将使用默认后端的实例。有关 更多详细信息，请参阅电子邮件后端的文档。*
- *html_message：如果html_message被提供，所得到的电子邮件将是一个 多部分/替代电子邮件message作为 文本/无格式内容类型和html_message作为 text / html的内容类型。*



> 这里选择django自带的[send_mail](https://docs.djangoproject.com/en/2.1/topics/email/)，故下面的步骤是按此而言

1. settings中配置

```
# 发送邮件的setting设置

EMAIL_HOST = "smtp.qq.com"
EMAIL_PORT = 25
EMAIL_HOST_USER = "isk2y@qq.com"
EMAIL_HOST_PASSWORD = " "
EMAIL_USE_TLS= True
EMAIL_FROM = "isk2y@qq.com"
```

这边我吧邮箱配置放到了另一个脚本中，并计入了gitignore

```python
#!/usr/bin/env python
# -*- coding:utf-8 -*-
#    @Author:iSk2y

from random import Random
from users.models import EmailVerifyRecord
from django.core.mail import send_mail
from mxonline.settings import EMAIL_FROM


def random_str(random_len=8):
    """
    生成随机字符串
    :param random_len: 字符串长度，默认为8
    :return: 返回字符串
    """
    rd_str = ''
    chars = 'AaBbCcDdEeFfGgHhIiJjKkLlMmNnOoPpQqRrSsTtUuVvWwXxYyZz0123456789'
    length = len(chars) - 1
    random = Random()
    for i in range(random_len):
        rd_str += chars[random.randint(0, length)]
    return rd_str


# 注册发信
def send_mx_email(email, send_type='register'):
    """
    发送邮件函数 : 发送前将生成的str拼接成url放入数据库，到时候查询数据库是否存在url
    :param email: 邮箱
    :param send_type: 发送类型，默认是注册，还有是找回密码
    :return:
    """
    email_record = EmailVerifyRecord()
    code = random_str(20)  # 生成随机字符串，数据库中code最长为20
    email_record.code = code
    email_record.email = email
    email_record.send_type = send_type
    email_record.save()

    # 定义邮件的内容
    email_title = ''
    email_body = ''

    if send_type == 'register':
        email_title = '我的慕学网站 注册激活链接'
        email_body = '请点击下面的链接激活你的账号：http://127.0.0.1:8000/active/{0}'.format(code)
        # 使用Django内置函数完成邮件发送。四个参数：主题，邮件内容，从哪里发，接受者list
        send_status = send_mail(email_title, email_body, EMAIL_FROM, [email])
        if send_status:
            return True
        else:
            return False

```



暂时没有加入body模板之类的，先用简单的进行测试，后续再进行添加修改完善整个发信函数

对应的在view中添加了一个激活的视图函数ActiveUserView

```python
class ActiveUserView(View):
    """
    激活用户视图函数
    """
    def get(self, request, active_code):
        # 取出所有符合验证码的记录 如果查询不到 激活链接肯定是假的 直接返回404
        record_list = get_list_or_404(models.EmailVerifyRecord,code= active_code)
        msg = ''

        for record in record_list:
            email = record.email
            # 根据邮箱 去查用户表中 未激活的用户
            try:
                user = models.UserProfile.objects.get(email=email, is_active=False)
                user.is_active = True
                user.save()
                msg = '用户激活成功，请登录!'
            except models.UserProfile.DoesNotExist:
                msg = '你的账户已是激活状态！'

        return render(request, 'login.html', {'msg': msg})
```

并在LoginView中需要添加逻辑判断是否用户已经激活



# 找回密码

首次看视频和代码感觉到的两个问题：（好像后面有review）

- 重复输入密码应该放在 forms is_valid中，自写clean函数进行检测
- 根据前端传入的email，明显存在任意密码重置漏洞

还要考虑重置密码链接是否被**点击过，过期时间**等！！！

## 设计url

```python
path('forget/', ForgetPwdView.as_view(), name='forget_pwd'),
# 找回密码url
re_path('reset/(?P<reset_code>.*)/', ResetView.as_view(), name='reset_pwd'),
# 后台修改密码操作的url
path('modify_pwd/', ModifyPwdView.as_view(), name='modify_pwd')
```

其实我觉得应该把这些都放入users这个app的urls内，然后include下，



## 设计对应form

form我认为只用来做校验比较好，不要直接拿到前端用，因为前端太多样式css了，在form里面写widget会累死的

```python
class ForgetForm(forms.Form):
    # 此处email与前端name需保持一致。
    email = forms.EmailField(required=True)
    # 应用验证码 自定义错误输出key必须与异常一样
    captcha = CaptchaField(error_messages={"invalid": u"验证码错误"})


class ModifyPwdForm(forms.Form):
    # 提交新密码的form
    password = forms.CharField(required=True, min_length=6, error_messages={
        'min_length': '最少为6个字符',
        'required': '必须输入密码啊！'
    })
    # 再次输入密码的字段
    re_pwd = forms.CharField(required=True)

    def clean_re_pwd(self):
        pwd = self.cleaned_data.get('password')
        re_pwd = self.cleaned_data.get('re_pwd')
        if pwd is None:
            raise ValidationError("请先正确输入密码！!")
        if pwd != re_pwd:
            raise ValidationError("两次密码不一样！！")
        return re_pwd

```

我在这里利用了它本身的hook机制 重载了re_pwd的clean方法，验证两次密码是否一致。不放在view中去做验证了

## 完善发信

```python
def send_mx_email(email, send_type='register'):
    """
    发送邮件函数 : 发送前将生成的str拼接成url放入数据库，到时候查询数据库是否存在url
    :param email: 邮箱
    :param send_type: 发送类型，默认是注册，还有是找回密码
    :return:
    """
    email_record = EmailVerifyRecord()
    code = random_str(20)  # 生成随机字符串，数据库中code最长为20
    email_record.code = code
    email_record.email = email
    email_record.send_type = send_type
    email_record.save()

    # 定义邮件的内容
    email_title = ''
    email_body = ''

    if send_type == 'register':
        email_title = '我的慕学网站 注册激活链接'
        email_body = '请点击下面的链接激活你的账号：http://127.0.0.1:8000/active/{0}'.format(code)
        # 使用Django内置函数完成邮件发送。四个参数：主题，邮件内容，从哪里发，接受者list

    elif send_type == 'forget':
        email_title = '我的慕学网站 密码找回链接'
        email_body = '请点击下面的链接重置密码：http://127.0.0.1:8000/reset/{0}'.format(code)

    send_status = send_mail(email_title, email_body, EMAIL_FROM, [email])
    if send_status:
        return True
    else:
        return False
```

## 视图View

按照视频中的思路写了3个视图类，但是我感觉有点臃肿，后面考虑是否重写

```python
class ForgetPwdView(View):
    """
    忘记密码显示页面
    """
    def get(self, request):
        forget_from = forms.ForgetForm()
        return render(request, "forgetpwd.html", {"forget_from": forget_from})

    def post(self, request):
        forget_form = forms.ForgetForm(request.POST)
        # form验证合法情况下取出email,从clean data中取
        if forget_form.is_valid():
            email = forget_form.cleaned_data.get("email", "")
            # 发送找回密码邮件
            send_mx_email(email, "forget")
            # 发送完毕返回登录页面并显示发送邮件成功。
            return render(request, "login.html", {"msg": "重置密码邮件已发送,请注意查收"})
        # 如果表单验证失败也就是他验证码输错等。
        else:
            return render(request, "forgetpwd.html", {"forget_from": forget_form})


class ResetView(View):
    """
    重置密码显示页面函数
    """
    def get(self, request, reset_code):
        # 查询生成的随机字符串 和对应的email
        record_list = get_object_or_404(models.EmailVerifyRecord, code=reset_code)

        return render(request, 'password_reset.html', {'reset_code': reset_code})


class ModifyPwdView(View):
    """
    修改密码操作视图
    """
    def post(self, request):
        modifypwd_form = forms.ModifyPwdForm(request.POST)
        if modifypwd_form.is_valid():
            # 两次密码是否一致验证已经在form检测中 验证
            password = modifypwd_form.cleaned_data.get("password")
            # 从前端取到resetcode这样定位到是谁在重置密码
            reset_code = request.POST.get("reset_code")
            # 如果重置reset code是伪造的 取不到object 直接给他返回404 干嘛还给他友好显示！！！
            email_obj = get_object_or_404(models.EmailVerifyRecord, code=reset_code)
            # 从记录验证码表中取出对应的email 去寻找user
            user = models.UserProfile.objects.get(email=email_obj.email)
            user.password = make_password(password)
            user.save()
            return render(request, 'login.html',{'msg':'密码重置成功！请登录'})
        else:
            reset_code = request.POST.get("reset_code")
            return render(request, 'password_reset.html', {'reset_code': reset_code, 'modifypwd_form':modifypwd_form})

```



## 补充

视频中没有提及的个人认为补充下：

1. 前端回传forms内容时可以不用if，error显示和value显示也好，感觉直接用filter过滤器的default更方便简单些

```python
{{ modifypwd_form.password.errors.0 |default:'' }} # 错误
{{ register_form.password.value |default:''}} # 值
```

2. password_reset模板中的标签
   - `<input type="submit" value="提交" >`最好修改成这样，onclick可能会触发失败的
   - 错误可以回显在`<i></i>`中，这里教程中好像没有提及
3. 应该还要添加链接是否点击，是否过期等考虑项



# 机构列表

终于开始采用模板继承的方式。

这里记录下MEDIA的配置方式：

- `MEDIA_ROOT`：在settings中配置，这样上传的文件会相对于这个路径。否则会相对于项目路径的
- `MEDIA_URL`：可以使得在模板中动态使用media，便于后期维护。记得先再setting的template中加入`'django.template.context_processors.media'`才能用
- 要在URL中配置让media访问的url，下面的方法仅仅**推荐在本地开发环境中，生产环境千万不要这样**

```python
from django.views.static import serve
from mxonline.settings import MEDIA_ROOT
from django.urls import path, include, re_path


urlpatterns = [
# 设置media访问url
    re_path(r'^media/(?P<path>.*)', serve, {"document_root": MEDIA_ROOT})
]
```



## 分页功能

这里分页功能选择了GitHub上一个Django的第三方包 [django-pure-pagination](https://github.com/jamespacileo/django-pure-pagination)

配合我们的例子使用是这样：

```python
class OrgListView(View):
    def get(self, request):
        org_list = models.CourseOrg.objects.all()
        citys_list = models.CityDict.objects.all()
        org_sum = models.CourseOrg.objects.count()

        # 尝试获取前台get请求传递过来的page参数
        # 如果是不合法的配置参数默认返回第一页
        try:
            page = request.GET.get('page', 1)
        except PageNotAnInteger as e:
            page = 1
        p = Paginator(org_list, 5, request=request)
        orgs = p.page(page)
        return render(request, 'org_list.html', {
            'org_list': orgs,
            'citys_list': citys_list,
            'org_sum': org_sum
        }
        )
```

模板中使用原样式，然后添加一些判断和遍历，不建议直接使用内置的样式，或者可以自己魔改他的模板

```html
<div class="pageturn">
    <ul class="pagelist">
        {% if org_list.has_previous %}
        <li class="long"><a href="?{{ org_list.previous_page_number.querystring }}">上一页</a></li>
        {% endif %}
        {% for page in org_list.pages %}
            {% if page %}
                {% ifequal page org_list.number %}
                <li class="active"><a href="?{{ page.querystring }}">{{ page }}</a></li>
                 {% else %}
                <li><a href="?{{ page.querystring }}" class="page">{{ page }}</a></li>
                {% endifequal %}
            {% else %}
            ...
            {% endif %}
        {% endfor %}
        {% if org_list.has_next %}
        <li class="long"><a href="?{{ org_list.next_page_number.querystring }}">下一页</a></li>
        {% endif %}
    </ul>
</div>
```



## 分类筛选

对模板中的 city_id category sort 参数在view中进行filter筛选，返回结果org_list 然后显示内容



## ”我要学习“部分

在这里使用modelForm了，就是把form和model结合在一起，form验证完可以直接插入model内，更加方便快捷。

```python
#!/usr/bin/env python
# -*- coding:utf-8 -*-
#    @Author:iSk2y
import re
from django import forms

from operation.models import UserAsk


class UserAskForm(forms.ModelForm):
    # 继承model外也能新增其他字段
    class Meta:
        model = UserAsk
        # 指定验证的字段
        fields = ['name', 'mobile', 'course_name']

    def clean_mobile(self):
        mobile = self.cleaned_data['mobile']
        REGEX_MOBILE = "^1[358]\d{9}$|^147\d{8}$|^176\d{8}$"
        p = re.compile(REGEX_MOBILE)
        if p.match(mobile):
            return mobile
        else:
            raise forms.ValidationError(u"手机号码非法", code="mobile_invalid")
```

在这里我直接修改了前端模板中的js代码，

```html
<script>
    $(function(){
        $('#jsStayBtn').on('click', function(){
            $.ajax({
                cache: false,
                type: "POST",
                url:"/org/add_ask/",
                data:$('#jsStayForm').serialize(),
                dataType: 'JSON', # 返回自动反序列化
                async: true,
                success: function(data) {
                    if(data.status == 'success'){
                        $('#jsStayForm')[0].reset();
                        $('#jsCompanyTips').html(data.msg)
                        alert("提交成功")
                    }else if(data.status == 'fail'){
                        $('#jsCompanyTips').html(data.msg)
                    }
                }
            });
        });
    })

</script>
```

视图里简单这样写了

```python
class AddUserAskView(View):
    """
    我要学习模块
    """
    def post(self, request):
        userask_form = forms.UserAskForm(request.POST)
        if userask_form.is_valid():
            # 如果验证通过 就存入数据库中
            userask_form.save(commit=True)
            return JsonResponse({"status": "success", "msg": "咨询成功，请等待回复"})
        else:
            return JsonResponse({"status": "fail", "msg": '咨询失败，稍后再试'})
```





## 机构详情

机构详情分为四个部分：主页、课程、介绍、教师

都是赋值模板 遍历的东西，重复做就行了，不过这边的URL我与视频中设计不同。我觉得他那样有点冗余，所以我分发了一下

```python
#!/usr/bin/env python
# -*- coding:utf-8 -*-
#    @Author:iSk2y

from django.urls import path, include, re_path
from organization import views


app_name = 'org'


extra_patterns = [
    path('home/', views.OrgHomeView.as_view(), name='org_home'),
    path('course/', views.OrgCourseView.as_view(), name='org_course'),
    path('desc/', views.OrgDescView.as_view(), name='org_desc'),
    path('teacher/', views.OrgTeacherView.as_view(), name='org_teacher'),

]

urlpatterns = [

    path('list/', views.OrgListView.as_view(), name='org_list'),
    path('add_ask/', views.AddUserAskView.as_view(), name='add_ask'),
    path('show/<int:org_id>/', include(extra_patterns)), # org_id一致 url分发一下 减少冗余


]
```



# 公开课



## 课程列表

模板还是一样的继承，变量赋值循环，不过这里get一个小知识点，CharField中的choice可以用特定方法显示为中文

```html
<div class="right layout">
    <div class="head">热门课程推荐</div>
    <div class="group_recommend">
        {% for course in hot_courses %}
        <dl>
            <dt>
            <a target="_blank" href="">
            <img width="240" height="220" class="scrollLoading"
            src="{{ MEDIA_URL }}{{ course.image }}"/>
            </a>
            </dt>
            <dd>
            <a target="_blank" href=""><h2> {{ course.name }}</h2></a>
            <span class="fl">难度：<i class="key">{{ course.get_degree_display }}</i></span>
            </dd>                               # 这里get_degree_display
        </dl>
        {% endfor %}
    </div>
</div>
```

视图简单这样写

```python
class CourseListView(View):
    """
    课程列表页面
    """
    def get(self, request):
        all_course = models.Course.objects.all()
        hot_courses = models.Course.objects.all().order_by("-students")[:3]
        # 对课程进行分页
        # 尝试获取前台get请求传递过来的page参数
        # 如果是不合法的配置参数默认返回第一页
        try:
            page = request.GET.get('page', 1)
        except PageNotAnInteger:
            page = 1
        # sort排序
        sort = request.GET.get('sort', "")
        if sort:
            if sort == "students":
                all_course = all_course.order_by("-students")
            elif sort == "hot":
                all_course = all_course.order_by("-click_nums")

        p = Paginator(all_course, 6, request=request)
        courses = p.page(page)
        return render(request, 'course-list.html', {
            'all_course': courses,
            'sort': sort,
            "hot_courses": hot_courses
        })
```

分页功能前端模板直接照搬org_list页面就行了，注意变量名修改下

## 详情页面

详情页中有些前端模板中需要显示的内容，需要后端对象用自己的外键跨表查询，通常一般我View视图内，但是这次看到教程中直接写在了model内。然后还是一样的render页面和变量。直接在模板中使用该model的方法就行了。感觉不错！

下面以Course为例，model修改为

```python
class Course(models.Model):
    """
    课程model
    """
    DEGREE_CHOICES = (
        ('cj', '初级'),
        ('zj', '中级'),
        ('gj', '高级')

    )
    name = models.CharField(max_length=50, verbose_name="课程名")
    desc = models.CharField(max_length=300, verbose_name="课程描述")
    detail = models.TextField(verbose_name="课程详情")
    learn_times = models.IntegerField(default=0, verbose_name='学习时长（分钟）')  # 分钟为单位
    course_org = models.ForeignKey(to=CourseOrg, verbose_name='所属机构', on_delete=models.CASCADE, null=True, blank=True)
    students = models.IntegerField(default=0, verbose_name='学习人数')
    category = models.CharField(verbose_name='课程类别', max_length=20, default='编程开发')
    tag = models.CharField(verbose_name='课程标签',default='', max_length=10)
    fav_nums = models.IntegerField(default=0, verbose_name='收藏人数')
    image = models.ImageField(upload_to='courses/%Y/%m', verbose_name='封面图', max_length=100)
    click_nums = models.IntegerField(default=0, verbose_name='点击数')
    add_time = models.DateTimeField(default=datetime.now, verbose_name='添加时间')
    degree = models.CharField(verbose_name='课程难度', choices=DEGREE_CHOICES, max_length=2, default='cj')

    class Meta:
        verbose_name = '课程'
        verbose_name_plural = verbose_name

    def get_lesson_nums(self):
        """
        获取章节数
        :return: int
        """
        return self.lesson_set.all().count()

    def get_learn_user(self):
        """
        获取学习该课程的5个用户
        :return: QurySet
        """
        return self.usercourse_set.all()[:5]

    def __str__(self):
        return self.name
```

在模板中直接这样使用了

```html
<div class="des">
<h1 title="{{ course.name }}">{{ course.name }}</h1>
<span class="key">{{ course.desc }}</span>
    <div class="prize">
        <span class="fl">难度：<i class="key">{{ course.get_degree_display }}</i></span>
        <span class="fr">学习人数：{{ course.students }}</span>
    </div>
    <ul class="parameter">
    <li><span class="pram word3">时&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;长：</span><span>{{ course.learn_times }}</span></li>
    <li><span class="pram word3">章&nbsp;节&nbsp;数：</span><span>{{ course.get_lesson_nums }}</span></li>
    <li><span class="pram word3">课程类别：</span><span title="">{{ course.category }}</span></li>
    <li class="piclist"><span class="pram word4">学习用户：</span>
    {% for usercourse in course.get_learn_user %}
    <span class="pic"><img width="40" height="40" src="/static/{{ usercourse.user.image }}"/></span>

    {% endfor %}


    </li>
    </ul>
    <div class="btns">
    <div class="btn colectgroupbtn"  id="jsLeftBtn">
    收藏
    </div>
    <div class="buy btn"><a style="color: white" href="course-video.html">开始学习</a></div>
    </div>
</div>
```

View中只这样写

```python
class CourseDetailView(View):
    """
    详情页
    """
    def get(self, request, course_id):
        course_obj = get_object_or_404(models.Course, pk=course_id)
        course_obj.click_nums += 1
        course_obj.save()
        return render(request, 'course-detail.html', {
            'course': course_obj
        })

```

## 章节信息

课程和章节是一对多关系，章节和视频也是一对多关系。

在视频中新添加url字段。给vedio添加一个学习时长的字段。然后添加模拟数据。

- 在course中添加获取章节的方法
- 在lesson中添加获取所有视频的方法
- 创建课程与讲师直接的外键关联

```html
<div class="mod-chapters">
{% for lesson in course.get_course_lesson %}
    <div class="chapter chapter-active" >
        <h3>
            <strong><i class="state-expand"></i>{{ lesson.name }}</strong>
        </h3>
        <ul class="video">
        {% for vedio in lesson.get_lesson_vedio %}
            <li>
            <a target="_blank" href='{{ vedio.url }}' class="J-media-item studyvideo">{{ vedio.name }}
            <i class="study-state"></i>
            </a>
            </li>
        {% endfor %}
        </ul>
    </div>
{% endfor %}

</div>
```

前端模板中for嵌套就形成了层级界面

## 课程评论

评论页面模板类似继承，然后添加评论也是ajax。

## 相关课程推荐

这里要通过orm去找 当前学习当前课程的用户还学过的其他课程 

```python
# 选出学了这门课的学生关系
user_courses = UserCourse.objects.filter(course= course)
# 从关系中取出user_id
user_ids = [user_course.user_id for user_course in user_courses]
# 这些用户学了的课程,外键会自动有id，取到字段
all_user_courses = UserCourse.objects.filter(user_id__in=user_ids)
# 取出所有课程id
course_ids = [all_user_course.course_id for all_user_course in all_user_courses]
# 获取学过该课程用户学过的其他课程
relate_courses = Course.objects.filter(id__in=course_ids).order_by("-click_nums")[:5]
# 是否收藏课程
return render(request, "course-video.html", {
"course": course,
"all_resources": all_resources,
"relate_courses":relate_courses,
})
```

然后还要把这个添加到三个视图中，我觉得这样代码重复太多了，考虑其他方式减少冗余。

## 使用include_tag

考虑到课程详情章节页面和评论页面，右边部分内容重复，我自己把他单拿出来放到了[include_tag](https://docs.djangoproject.com/zh-hans/2.1/howto/custom-template-tags/#inclusion-tags)里面了。这样减少代码重复度。记录下简要步骤

> 在APP下建立templatetags包文件然后新建tag文件，这里以get_content.py为例
>
> 在tag文件中创建定义函数，
>
> 这里定义takes_context=True，可以自动获取上下文变量。就是在模板中的变量可以从context这个字典中取出来

```python
#!/usr/bin/env python
# -*- coding:utf-8 -*-
#    @Author:iSk2y


from django import template
from courses import models
from operation.models import UserCourse
from mxonline.settings import MEDIA_URL
register = template.Library()


@register.inclusion_tag('course_aside.html', takes_context=True)
def get_right_side(context):
    # 从上下文变量中取出course
    course = context['course']
    all_resources = models.CourseResource.objects.filter(course=course)
    # 选出学了这门课的学生
    user_courses = UserCourse.objects.filter(course=course)
    # 从关系中取出user_id遍历成列表
    user_ids = [user_course.user_id for user_course in user_courses]
    # 查询出用户课程表中 用户id在上面列表中的 其他课程
    all_user_courses = UserCourse.objects.filter(user_id__in=user_ids)
    # 取出所有课程id
    course_ids = [all_user_course.course_id for all_user_course in all_user_courses]
    # 获取学过该课程用户学过的其他课程
    relate_courses = models.Course.objects.filter(id__in=course_ids).order_by("-click_nums")[:5]
    # 是否收藏课程

    return {
        'all_resources': all_resources,
        'course': course,
        # 我也不知道为啥 这个MEDIA 不传前端就获取不到了 那就显式传递一下吧
        'MEDIA_URL': MEDIA_URL,
        "relate_courses": relate_courses,
    }

```

这个是get_right_side中使用的模板

```html
<div class="aside r">
    <div class="bd">

        <div class="box mb40">
            <h4>资料下载</h4>
            <ul class="downlist">
                {% for resource in all_resources %}
                    <li>
                        <span><i class="aui-iconfont aui-icon-file"></i>&nbsp;&nbsp;{{ resource.name }}</span>
                        <a href="{{ MEDIA_URL }}{{ resource.download }}" class="downcode" target="_blank" download=""
                           data-id="274" title="">下载</a>
                    </li>
                {% endfor %}


            </ul>
        </div>
        <div class="box mb40">
            <h4>讲师提示</h4>
            <div class="teacher-info">
                <a href="/u/315464/courses?sort=publish" target="_blank">
                    <img src='{{ MEDIA_URL }}{{ course.teacher.image }}' width='80' height='80'/>
                </a>
                <span class="tit">
          <a href="/u/315464/courses?sort=publish" target="_blank">{{ course.teacher.name }}</a>
        </span>
                <span class="job">{{ course.teacher.work_position }}</span>
            </div>
            <div class="course-info-tip">
                <dl class="first">
                    <dt>课程须知</dt>
                    <dd class="autowrap">{{ course.need_know }}</dd>
                </dl>
                <dl>
                    <dt>老师告诉你能学到什么？</dt>
                    <dd class="autowrap">{{ course.teacher_tell }}</dd>
                </dl>
            </div>
        </div>


        <div class="cp-other-learned  js-comp-tabs">
            <div class="cp-header clearfix">
                <h2 class="cp-tit l">该课的同学还学过</h2>
            </div>
            <div class="cp-body">
                <div class="cp-tab-pannel js-comp-tab-pannel" data-pannel="course" style="display: block">
                    <!-- img 200 x 112 -->
                    <ul class="other-list">
                        {% for course in relate_courses %}
                         <li class="curr">
                            <a href="{% url 'course:detail' course.id %}" target="_blank">
                                <img src="{{ MEDIA_URL }}{{ course.image }}"
                                     alt="{{ course.name }}">
                                <span class="name autowrap">{{ course.name }}</span>
                            </a>
                        </li>
                        {% endfor %}
                    </ul>
                </div>

            </div>
        </div>

    </div>
</div>
```

> 再到前面模板中使用这个include_tag

```django
{% load get_content %}

 {% get_right_side %}
```

这样就不用在course_comment，course_vedio页面两次使用相同内容的HTML代码，也不用在CourseInfoView，CommentsView视图里用同样的代码获取变量了。

# CBV装饰器

一些视图的操作必须登录才能进行，这样就要给某些视图类加上登录的判断。当然也可以用

```python
if request.user.is_authenticated:
```

但是这样判断后在进行操作总归觉得不是最好的解决方式。

所以这里改用装饰器。FBV模式直接用@login_required就行了。但是CBV模式的需要另外的方法

```python
# apps/utils/mixin_utils.py

from django.contrib.auth.decorators import login_required
from django.utils.decorators import method_decorator

class LoginRequiredMixin(object):
    @method_decorator(login_required(login_url='/login/'))   
    def dispatch(self,request,*args,**kwargs):
        return super(LoginRequiredMixin, self).dispatch(request,*args,**kwargs)
```

在django中已Mixin结尾的，就代表最基本的View

然后让CourseInfoView和CommentsView都继承LoginRequiredMixin



# 主页

重新继承自base，之前开始写的时候很随便，直接用的index.html 现在要考虑整体性，所以教程中又继承自base了。其他操作没有什么特殊，就是替换render而已。这里有个对主页导航栏active判断的方法之前没有想到过，记录下

配置全局导航和全局"active"状态

```html
<div class="wp">
    <ul>
    <li {% if request.path == '/' %}class="active"{% endif %}><a href="{% url 'index' %}">首页</a></li>
    <li {% if request.path|slice:'7' == '/course' %}class="active"{% endif %}>
    <a href="{% url 'course:course_list' %}">
    公开课<img class="hot" src="{% static 'images/nav_hot.png' %}">
    </a>
    </li>
    <li {% if request.path|slice:'12' == '/org/teacher' %}class="active"{% endif %}>
    <a href="{% url 'org:teacher_list' %}">授课教师</a>
    </li >
    <li {% if request.path|slice:'9' == '/org/list' %}class="active"{% endif %}>
    <a href="{% url 'org:org_list' %}">授课机构</a></li>
    </ul>
</div>
```

说明：

- request.path  可以获取当前访问页面的相对url
- 比如“http://127.0.0.1:8000/org/teacher_list/”，则request.path  就是“/org/teacher_list/”
- 比如"http://127.0.0.1:8000/org/teacher/detail/1/"，则request.path 就是“/org/teacher/detail/1/”
- slice:12   是过滤器，取前七位数
- 利用这种发发可以达到全局的“active”效果，而不用每个子页面都要去设置“active”了

## 全局搜索功能

先来看看deco-common.js中的代码

```javascript
//顶部搜索栏搜索方法
function search_click(){
    var type = $('#jsSelectOption').attr('data-value'),
        keywords = $('#search_keywords').val(),
        request_url = '';
    if(keywords == ""){
        return
    }
    if(type == "course"){
        request_url = "/course/list?keywords="+keywords
    }else if(type == "teacher"){
        request_url = "/org/teacher/list?keywords="+keywords
    }else if(type == "org"){
        request_url = "/org/list?keywords="+keywords
    }
    window.location.href = request_url
}

```

在对应的视图中取keywords这个键值，然后使用orm操作sql like方法找到结果，再正常返回页面，这里仅以Course为例

```python

# 搜索功能
        search_keywords = request.GET.get('keywords', '')
        if search_keywords:
            # Q可以实现多个字段，之间是or的关系
            all_course = all_course.filter(
                Q(name__icontains=search_keywords) |
                Q(desc__icontains=search_keywords) |
                Q(detail__icontains=search_keywords)
            )
```

Q方法是or的用法



# 个人信息中心

个人资料的修改这部分是后面逻辑性较强的一块了。

## 修改头像

使用modelform，然后用meta指定field只为image，上传后直接保存



## 修改密码

使用modelform，再自写其中的clean方法，对两次输入的密码做相同比对，就是使用的上次的modifyform

```python
class ModifyPwdForm(forms.Form):
    # 提交新密码的form
    password = forms.CharField(required=True, min_length=6, error_messages={
        'min_length': '最少为6个字符',
        'required': '必须输入密码啊！'
    })
    # 再次输入密码的字段
    re_pwd = forms.CharField(required=True)

    def clean_re_pwd(self):
        pwd = self.cleaned_data.get('password')
        re_pwd = self.cleaned_data.get('re_pwd')
        if pwd is None:
            raise ValidationError("请先正确输入密码！!")
        if pwd != re_pwd:
            raise ValidationError("两次密码不一样！！")
        return re_pwd
```

view里面这样写

```python
class UpdatePwdView(LoginRequiredMixin, View):
    """
    个人中心修改用户密码
    """
    def post(self, request):
        modify_form = ModifyPwdForm(request.POST)
        # 我把两次密码是否相同验证放在clean方法内了
        if modify_form.is_valid():
            password = modify_form.cleaned_data.get("password")
            user = request.user
            user.password = make_password(password)
            user.save()

            return HttpResponse('{"status":"success"}', content_type='application/json')
        else:
            # 两次密码不一致会在这里返回的
            return HttpResponse(json.dumps(modify_form.errors), content_type='application/json')
```

## 修改邮箱

修改邮箱属于这里面逻辑更多的一部分

- 判断要修改的邮箱是否已经存在，不存在就发送邮件验证码
- 判断验证码是否正确（去验证码表中寻找和当前用户比对）

```python
class SendEmailCodeView(LoginRequiredMixin, View):
    """
    发送邮箱验证码视图
    """
    def get(self, request):
        email = request.GET.get('email', '')

        if UserProfile.objects.filter(email=email):
            return HttpResponse('{"email":"邮箱已存在"}', content_type='application/json')

        send_mx_email(email, 'update_email')
        return HttpResponse('{"status":"success"}', content_type='application/json')


class UpdateEmailView(LoginRequiredMixin, View):
    """
    修改邮箱视图
    """
    def get(self, request):
        email = request.POST.get("email", "")
        code = request.POST.get("code", "")

        existed_records = models.EmailVerifyRecord.objects.filter(email=email, code=code, send_type='update_email')
        if existed_records:
            user = request.user
            user.email = email
            user.save()
            return HttpResponse('{"status":"success"}', content_type='application/json')
        else:
            return HttpResponse('{"email":"验证码无效"}', content_type='application/json')
```



## 查看收藏和信息

个人中心还可以查看收藏（机构、教师、课程）、消息、和自己学习的公开课程。还是一样的查然后赋值给模板渲染。

```python
class MyCourseView(LoginRequiredMixin, View):
    def get(self, request):
        user_courses = opmodel.UserCourse.objects.filter(user=request.user)
        return render(request, "usercenter-mycourse.html", {
            "user_courses": user_courses,
        })


class MyFavOrgView(LoginRequiredMixin, View):
    """
    我收藏的课程机构
    """

    def get(self, request):
        org_list = []
        fav_orgs = opmodel.UserFavorite.objects.filter(user=request.user, fav_type=2)
        # 上面的fav_orgs只是存放了id。我们还需要通过id找到机构对象
        for fav_org in fav_orgs:
            # 取出fav_id也就是机构的id。
            org_id = fav_org.fav_id
            # 获取这个机构对象
            org = CourseOrg.objects.get(id=org_id)
            org_list.append(org)
        return render(request, "usercenter-fav-org.html", {
            "org_list": org_list,
        })


class MyFavTeacherView(LoginRequiredMixin, View):
    """
    我收藏的授课讲师
    """
    def get(self, request):
        teacher_list = []
        fav_teachers = opmodel.UserFavorite.objects.filter(user=request.user, fav_type=3)
        for fav_teacher in fav_teachers:
            teacher_id = fav_teacher.fav_id
            teacher = Teacher.objects.get(id=teacher_id)
            teacher_list.append(teacher)
        return render(request, "usercenter-fav-teacher.html", {
            "teacher_list": teacher_list,
        })


class MyFavCourseView(LoginRequiredMixin, View):
    """
    我收藏的公开课程
    """
    def get(self, request):
        course_list = []
        fav_courses = opmodel.UserFavorite.objects.filter(user=request.user, fav_type=1)
        for fav_course in fav_courses:
            course_id = fav_course.fav_id
            course = Course.objects.get(id=course_id)
            course_list.append(course)

        return render(request, 'usercenter-fav-course.html', {
            "course_list": course_list,
        })


class MyMessageView(LoginRequiredMixin, View):
    def get(self, request):
        all_message = opmodel.UserMessage.objects.filter(user=request.user.id)

        try:
            page = request.GET.get('page', 1)
        except PageNotAnInteger:
            page = 1
        p = Paginator(all_message, 4, request=request)
        messages = p.page(page)
        return render(request, "usercenter-message.html", {
            "messages": messages,
        })
```



> 在页面导航栏都会有消息个数提醒，那如何把消息个数变量放置在每个页面中？

考虑到每个页面中的request.user本来就不用显式赋值给模板调用，所以可以直接在userProfile的model中加入一个获取消息个数的方法！！Mark 之前都未想到过这样使用。

# 细节收尾

对各种点击数保存在视图中添加相应代码，对收藏操作进行一定判断。对导航栏class是否有active状态进行判断，

```html
<div class="left">
            <ul>
                <li {% ifequal  '/user/info/' request.path %} class="active2" {% endifequal %}><a href="{% url 'user:user_info' %}">个人资料</a></li>
                <li {% ifequal  '/user/mycourse/' request.path %} class="active2" {% endifequal %}><a href="{% url 'user:mycourse' %}">我的课程</a></li>
                <li {% ifequal  '/user/myfav/' request.path|slice:'12' %} class="active2" {% endifequal %}><a href="{% url 'user:myfav_org' %}">我的收藏</a></li>
                <li {% ifequal  '/user/my_message/' request.path %} class="active2" {% endifequal %}>
                    <a href="{% url 'user:my_message' %}" style="position: relative;">
                        我的消息
                    </a>
                </li>
            </ul>
        </div>
```

这里以usercenter-base为例，这样处理判断active的。

# 全局配置404和500

视频中讲到了配置需要注意的点：

## handler

在urls中定义handler404和handler500

```python
# 全局404页面配置
handler404 = 'users.views.page_not_found'
# 全局500页面配置
handler500 = 'users.views.page_error'
```

page_not_found和page_error两个函数定义在视图中，404和500时执行对应的操作

## 定义处理函数

```python
from django.shortcuts import render_to_response
def pag_not_found(request):
    # 全局404处理函数
    response = render_to_response('404.html', {})
    response.status_code = 404
    return response

def page_error(request):
    # 全局500处理函数
    from django.shortcuts import render_to_response
    response = render_to_response('500.html', {})
    response.status_code = 500
    return response
```



## 关闭DEBUG

如果不关闭debug模式，不会执行我们指定的404和500函数内的代码，而且仍旧报错，这是由于Django的调试模式决定的



## 设置static访问

视频中说到，关闭debug模式后，Django不会接管我们的静态文件，需要和media一样设置一个url路由给它访问。

```python
re_path(r'^static/(?P<path>.*)', serve, {"document_root": STATIC_ROOT }),
```

一般静态文件通过第三方http服务器代理转发。nignx 和 Apache都会自动代理这些静态文件



## allowed_hosts

settings中的allowed_hosts设置项为允许运行该站点的ip，暂时先设置为*。



# 富文本编辑器

教程中使用了DjangoUeditor。这里记录一下添加这个组件的过程。

## 下载

https://github.com/twz915/DjangoUeditor3/

下载后放置随意一个目录下，我放于extra_apps下

## 配置

> settings中添加app

```python
INSTALLED_APPS = [
    'DjangoUeditor',
]
```



> mxonline/urls.py


```python
# 富文本编辑器url
    path('ueditor/',include('DjangoUeditor.urls' )),
```



> course/models.py中Course修改detail字段


```python
from DjangoUeditor.models import UEditorField

class Course(models.Model):
    # detail = models.TextField("课程详情")
    detail = UEditorField(verbose_name=u'课程详情', width=600, height=300, imagePath="courses/ueditor/",
                          filePath="courses/ueditor/", default='')
```



> xadmin/plugs目录下新建ueditor.py文件

```python
import xadmin
from xadmin.views import BaseAdminPlugin, CreateAdminView, ModelFormAdminView, UpdateAdminView
from DjangoUeditor.models import UEditorField
from DjangoUeditor.widgets import UEditorWidget
from django.conf import settings


class XadminUEditorWidget(UEditorWidget):
    def __init__(self, **kwargs):
        self.ueditor_options = kwargs
        self.Media.js = None
        super(XadminUEditorWidget,self).__init__(kwargs)


class UeditorPlugin(BaseAdminPlugin):

    def get_field_style(self, attrs, db_field, style, **kwargs):
        if style == 'ueditor':
            if isinstance(db_field, UEditorField):
                widget = db_field.formfield().widget
                param = {}
                param.update(widget.ueditor_settings)
                param.update(widget.attrs)
                return {'widget':XadminUEditorWidget(**param)}
        return attrs

    def block_extrahead(self, context, nodes):
        js  = '<script type="text/javascript" src="%s"></script>' %(settings.STATIC_URL + "ueditor/ueditor.config.js")
        js += '<script type="text/javascript" src="%s"></script>' %(settings.STATIC_URL + "ueditor/ueditor.all.min.js")
        nodes.append(js)

xadmin.site.register_plugin(UeditorPlugin, UpdateAdminView)
xadmin.site.register_plugin(UeditorPlugin, CreateAdminView)
```



> `xadmin/plugs/__init__.py`里面添加ueditor插件

```python
PLUGINS = (
   'ueditor',
)
```



> course/adminx.py中使用

```python
class CourseAdmin(object):
    #detail就是要显示为富文本的字段名
    style_fields = {"detail": "ueditor"}
```



> course-detail.html

必须要在需要显示富文本的地方关闭自动转义

```python
<div class="tab_cont tab_cont1">
     {% autoescape off %}
     {{ course.detail }}
     {% endautoescape %}
     </div>
```

课程中还有xadmin进阶开发和部署上线等步骤在此不展开了，代码开发全部完成。结束啦撒花！！！
下面说说收获吧！


# 收获



- CBV模式的进一步熟悉使用，CBV方式的装饰器
- model建立思想，一对多关系等。为防止循环调用，需要构建的有层次。
- 在model中书写某些orm查询，而后在模板中直接调用该变量的属性来使用。而不直接在view中书写
- 注册、找回密码、更换邮箱、全局消息等具体实现逻辑思路。
- xadmin组件的进阶使用
- 全局配置处理404和500的方法。
- 项目部署上线的大体步骤。



![慕学在线网.png](https://upload-images.jianshu.io/upload_images/14657587-dcff0b56d11449fe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
