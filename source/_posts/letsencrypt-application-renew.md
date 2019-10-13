---
title: 申请Let's encrypt的SSL证书
date: 2019-05-13 23:36:39
tags: 
 - ssl
categories: 工具使用
---

HTTPS必须申请SSL证书



免费的证书还是用Let‘s Encrypt，**有效期3个月要及时续签**

<!-- more -->

现在有很多工具可以申请Let’s Encrypt来签发证书。win和Linux平台都有



只要实现了acme协议的，就可以从lentsencrypt生成免费的证书



## 理解验证流程

申请证书必须要经过验证才能生效，一般验证的方式有：[官方工具certbot介绍](https://certbot.eff.org/docs/using.html#getting-certificates-and-choosing-plugins)

| Plugin                                                       | Notes                                                        | Challenge types (and port)                                   |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| [apache](https://certbot.eff.org/docs/using.html#apache)     | Automates obtaining and installing a certificate with Apache. | [http-01](https://tools.ietf.org/html/draft-ietf-acme-acme-03#section-7.2) (80) |
| [nginx](https://certbot.eff.org/docs/using.html#nginx)       | Automates obtaining and installing a certificate with Nginx. | [http-01](https://tools.ietf.org/html/draft-ietf-acme-acme-03#section-7.2) (80) |
| [webroot](https://certbot.eff.org/docs/using.html#webroot)   | Obtains a certificate by writing to the webroot directory ofan already running webserver. | [http-01](https://tools.ietf.org/html/draft-ietf-acme-acme-03#section-7.2) (80) |
| [standalone](https://certbot.eff.org/docs/using.html#standalone) | Uses a “standalone” webserver to obtain a certificate.Requires port 80 to be available. This is useful onsystems with no webserver, or when direct integration withthe local webserver is not supported or not desired. | [http-01](https://tools.ietf.org/html/draft-ietf-acme-acme-03#section-7.2) (80) |
| [DNS plugins](https://certbot.eff.org/docs/using.html#dns-plugins) | This category of plugins automates obtaining a certificate bymodifying DNS records to prove you have control over adomain. Doing domain validation in this way isthe only way to obtain wildcard certificates from Let’sEncrypt. | [dns-01](https://tools.ietf.org/html/draft-ietf-acme-acme-03#section-7.4) (53) |
| [manual](https://certbot.eff.org/docs/using.html#manual)     | Helps you obtain a certificate by giving you instructions toperform domain validation yourself. Additionally allows youto specify scripts to automate the validation task in acustomized way. | [http-01](https://tools.ietf.org/html/draft-ietf-acme-acme-03#section-7.2) (80) or [dns-01](https://tools.ietf.org/html/draft-ietf-acme-acme-03#section-7.4) (53) |



- webroot就是通过某个已经运行的web服务器上来验证
- standalone是通过他自己在80再起一个服务，然后来验证。所以得考虑之前80端口是否被占用，否则会冲突
- DNS 通过dns记录的方式来验证这个域名的所有权限。这种方式说白了，就是根据提示增加某个TXT记录的域名，然后记录值是一串验证字符。



从另一个角度来说支持三种验证方式

- dns-01：给域名添加一个 DNS TXT 记录。
- http-01：在域名对应的 Web 服务器下放置一个 HTTP well-known URL 资源文件。
- tls-sni-01：在域名对应的 Web 服务器下放置一个 HTTPS well-known URL 资源文件。





一般服务器如果域名解析到80端口正常，那就直接用webroot或者其他网站认证的方式来验证就行了。但是如果80端口不可用的情况下，就可以手动DNS验证的方式来解决这个情况。但是如果要申请通配符证书，只能使用dns-01的方式



## 工具



### cerbot

> 平台：Linux

使用文档：<https://certbot.eff.org/docs/using.html>



参考：《[Let's Encrypt 终于支持通配符证书了](https://www.jianshu.com/p/c5c9d071e395)》**具体手动验证的方式**



```
certbot certonly \
--email your-email@example.com \
--agree-tos \
--preferred-challenges dns \
--server https://acme-v02.api.letsencrypt.org/directory \
--manual \
-d yourdomain.com \
-d *.yourdomain.com
```



- ***certonly*** 获取或更新证书，但是不安装到本机。这个参数默认是run，即获取或更新证书并安装。另一个值是renew，即更新证书。
- ***--email*** 接收有关账户的重要通知的邮箱地址，非必要，建议最好带上
- ***--agree-tos*** 同意ACME服务器的订阅协议
- ***--preferred-challenges dns*** 以DNS Plugins的方式进行验证
- ***--server https://acme-v02.api.letsencrypt.org/directory*** 指定验证服务器地址为acme-v02的，因为默认的服务器地址是acme-v01的，不支持通配符验证
- ***--manual*** 采用手动交互式的方式验证
- ***-d yourdomain.com -d \*.yourdomain.com*** 指定要验证的域名。注意，不带www的一级域名yourdomain.com，和通配符二级域名*.yourdomain.com都要写，如果只写*.yourdomain.com生成出来的证书是无法识别yourdomain.com的



### acme.sh

多OS发行版本，在windows得安装cygwin支持curl OpenSSL crontab

<https://github.com/Neilpang/acme.sh>

中文说明：[https://github.com/Neilpang/acme.sh/wiki/%E8%AF%B4%E6%98%8E](https://github.com/Neilpang/acme.sh/wiki/说明)



具体方法在中文文档中说的很明白，需要查看即可。



[Windows使用acme.sh申请let’s encrypt泛域名SSL证书方法(附带自动续期)](<https://blog.853lab.com/2018/09/post_1965.html>)





## 自动验证（更新）

生成的免费ssl证书3个月就要重新renew一次。



[Let’s Encrypt通配符证书的申请与自动更新（附阿里云域名的HOOK脚本）](http://blog.dreamlikes.cn/archives/1028)

certbot的可以了解下这个教程内容



acme的可以了解上面的

[Windows使用acme.sh申请let’s encrypt泛域名SSL证书方法(附带自动续期)](<https://blog.853lab.com/2018/09/post_1965.html>)



