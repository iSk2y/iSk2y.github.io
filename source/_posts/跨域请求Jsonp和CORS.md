---
title: 跨域请求Jsonp和CORS
date: 2018.10.28 10:27:30
tags: JavaScript
categories: JavaScript
---



## 同源策略

同源策略（Same origin policy）是一种约定，它是浏览器最核心也最基本的安全功能，如果缺少了同源策略，则浏览器的正常功能可能都会受到影响。可以说Web是构建在同源策略基础之上的，浏览器只是针对同源策略的一种实现。

同源策略，它是由Netscape提出的一个著名的安全策略。现在所有支持JavaScript 的浏览器都会使用这个策略。**所谓同源是指，域名，协议，端口相同**。当一个浏览器的两个tab页中分别打开来 百度和谷歌的页面当浏览器的百度tab页执行一个脚本的时候会检查这个脚本是属于哪个页面的，即检查是否同源，只有和百度同源的脚本才会被执行。如果非同源，那么在请求数据时，浏览器会在控制台中报一个异常，提示拒绝访问。

<!-- more -->


### 同源限制

同源策略限制以下几种行为：

- Cookie、LocalStorage 和 IndexDB 无法读取
- DOM 和 Js对象无法获得
- AJAX 请求不能发送



### 常见跨域场景

```http
URL                                      说明                    是否允许通信
http://www.domain.com/a.js
http://www.domain.com/b.js         同一域名，不同文件或路径           允许
http://www.domain.com/lab/c.js

http://www.domain.com:8000/a.js
http://www.domain.com/b.js         同一域名，不同端口                不允许
 
http://www.domain.com/a.js
https://www.domain.com/b.js        同一域名，不同协议                不允许
 
http://www.domain.com/a.js
http://192.168.4.12/b.js           域名和域名对应相同ip              不允许
 
http://www.domain.com/a.js
http://x.domain.com/b.js           主域相同，子域不同                不允许
http://domain.com/c.js
 
http://www.domain1.com/a.js
http://www.domain2.com/b.js        不同域名                         不允许
```





## 情景示例

因为正好在学习Django，所以下面例子都用Django做为示例



站点2向站点1请求数据

> 站点1（称为demo1）

```python
#view.py

from django.shortcuts import render,HttpResponse

# Create your views here.


def showjson(request):
    import json
    data = {"status":True,"msg":"test"}
    return HttpResponse(json.dumps(data))

```

> 站点2（demo2）

```python

#views.py
from django.shortcuts import render,HttpResponse

# Create your views here.

def getjson(request):
    return render(request,'index.html')

--------------------------------------------------------------------
# index.html
<!DOCTYPE html>
<html lang="zh-cn">
<head>
    <meta charset="UTF-8">
    <title>index</title>
    <script src="https://cdn.bootcss.com/jquery/3.3.1/jquery.min.js"></script>
</head>
<body>
<button type="button">请求demo1站点</button>
<script>
    $("button").click(function () {
        $.ajax({
            url:'http://127.0.0.1:8000/showjson/',
            type:'GET',
        })
    })
</script>
</body>
</html>

```

点击button触发ajax，让demo2请求demo1的showjson，产生错误

> 谷歌

![谷歌](https://upload-images.jianshu.io/upload_images/14657587-66af312ba2cbc8d8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


> 火狐

![火狐](https://upload-images.jianshu.io/upload_images/14657587-a37203d5d7c1ecda.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


火狐的错误提示说得更加清楚



**注意：**这里要清楚，其实请求已经发出去了，只是在接收返回到浏览器时，由于同源策略，被拦截读取



## src属性

那同样是请求为何我们用jQuery的ajax去get请求会产生错误，而script的src引入却没有报跨域请求错误？

类似的标签还有 img，link，iframe和script。初步得出个结论，**有src属性的标签允许跨域**





## Jsonp

什么是`jsonp`？维基百科的定义是：`JSONP（JSON with Padding）`是资料格式 `JSON` 的一种“使用模式”。

`JSONP`也叫填充式JSON，是应用JSON的一种新方法，只不过是被包含在函数调用中的JSON。

原理是通过script标签的跨域特性来绕过同源策略。（需要服务器端配合，商量好）



### 原生实现



#### 固定函数名

script标签src其实就是引入目标文件中的内容，然后执行该代码。

- 我们先定义好func(name)这个函数。
- script的src引入时，执行该函数。代码执行成功，命令行打印出test字符串



> demo2站点 html文件

```html
<!DOCTYPE html>
<html lang="zh-cn">
<head>
    <meta charset="UTF-8">
    <title>index</title>
    
</head>
<body>


<script>
    function func(name) {
        console.log(name);
    }

</script>
<script src="http://127.0.0.1:8000/showjson/"></script>
</body>
</html>
```

> demo1站点view函数

```python
def showjson(request):
    
    return HttpResponse("%s(%s)" % ("func","'test'"))
```

这样不够灵活，实现性价比太低



#### 自定函数名

> demo2中HTML文件

```html
<!DOCTYPE html>
<html lang="zh-cn">
<head>
    <meta charset="UTF-8">
    <title>index</title>
    
</head>
<body>


<script>
    function func(name) {
        console.log(name);
    }

</script>
<script src="http://127.0.0.1:8000/showjson/?callback=func"></script>
</body>
</html>
```

> demo1的view

```python
from django.views.decorators.csrf import csrf_exempt
# Create your views here.

@csrf_exempt
def showjson(request):

    callback = request.GET.get("callback")
    return HttpResponse("%s(%s)" % (callback,"'test'"))
```

通过添加get参数callback动态设置函数名，前端使用一致函数名就可以了。

但是一载入就发get请求了，下面改写为点击触发



#### 模拟创建script

> demo2中HTML文件

```html
<!DOCTYPE html>
<html lang="zh-cn">
<head>
    <meta charset="UTF-8">
    <title>index</title>
    
</head>
<body>
<button type="button" onclick="toget()">请求demo1站点</button>

<script>
    function func(name) {
        console.log(JSON.stringify(name));
    }

    function toget() {
        var script = document.createElement("script");
        script.setAttribute("type","text/javascript");
        script.src = "http://127.0.0.1:8000/showjson/?callback=func";
        document.body.appendChild(script);
        document.body.removeChild(script);
    }
    
</script>

</body>
</html>
```

> demo1中函数

这次以发送json数据为例

```python
@csrf_exempt
def showjson(request):
    import json
    data = {"status":True,"msg":"test"}
    callback = request.GET.get("callback")
    return HttpResponse("%s(%s)" % (callback,json.dumps(data)))
```

这样就能正常获取json数据了



### jQuery实现

```javascript
<script>
    function toget() {
        $.ajax({
            url:'http://127.0.0.1:8000/showjson/',
            dataType:'jsonp',
            jsonp:'callback',
            jsonpCallback:'func'
        });
    }
    function func(arg) {
        console.log(JSON.stringify(arg))
    }
</script>
```

这种方式是自己指定回调执行函数，那直接用ajax success回调函数更简单



```javascript
<script>
    function toget() {
        $.ajax({
            url:'http://127.0.0.1:8000/showjson/',
            dataType:'jsonp',
            jsonp:'callback',
            success:function (arg) {
                console.log(JSON.stringify(arg))
            }
        });
    }
</script>
```

这种方式 jQuery自己生产了一个随机的callback参数值，去请求然后执行。例如上面这个例子的url

```
http://127.0.0.1:8000/showjson/?callback=jQuery33105892941255188728_1540691603147&_=1540691603148
```



## CORS

`CORS（Cross-Origin Resource Sharing`）跨域资源共享，定义了必须在访问跨域资源时，浏览器与服务器应该如何沟通。`CORS`背后的基本思想就是使用自定义的HTTP头部让浏览器与服务器进行沟通，从而决定请求或响应是应该成功还是失败。



普通跨域请求：只服务端设置Access-Control-Allow-Origin即可，前端无须设置，若要带cookie请求：前后端都需要设置



还是以上面最初错误跨域例子为例，只要在demo1被请求（服务端）返回的response响应头添加字段就行

> demo2中HTML

```html
<button type="button" onclick="toget()">请求demo1站点</button>

<script>
    function toget() {
        $.ajax({
            url:'http://127.0.0.1:8000/showjson/',
            success:function (data) {
                console.log(data)
            }
        });
    }
</script>
```

> demo1中view

```python
@csrf_exempt
def showjson(request):
    import json
    data = {"status":True,"msg":"test"}

    res = HttpResponse(json.dumps(data))
    res['Access-Control-Allow-Origin'] = 'http://127.0.0.1:8001'
    return res
```



## 深入资料参考

[前端常见跨域解决方案（全）](https://segmentfault.com/a/1190000011145364)

[跨域资源共享 CORS 详解](http://www.ruanyifeng.com/blog/2016/04/cors.html)

[同源策略与Jsonp](https://www.cnblogs.com/yuanchenqi/articles/7638956.html#_label6)

