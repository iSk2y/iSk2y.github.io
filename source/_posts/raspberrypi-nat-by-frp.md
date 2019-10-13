---
title: raspberry pi 内网穿透
date: 2017-04-30 23:42:58
tags: 
 - raspberrypi
 - frp
categories: raspberrypi
---

使用的是frp让树莓派内网穿透，顺便记录下

主要尝试过了几种工具，然后选择了frp

nat123 速度极慢 还有限制 还有套路 真的很套路

花生壳 免费版内网穿透只限电信网

<!-- more -->

## frp

> GitHub地址：<https://github.com/fatedier/frp>

这里注意 要下载对应版本 ，我树莓派使用的官方的系统 应该要下载 <https://github.com/fatedier/frp/releases/download/v0.9.3/frp_0.9.3_linux_arm.tar.gz> 否则配置完就编译不成功

## 通过自定义域名访问部署于内网的 web 服务

> 有时想要让其他人通过域名访问或者测试我们在本地搭建的 web 服务，但是由于本地机器没有公网 IP，无法将域名解析到本地的机器，通过 frp 就可以实现这一功能，以下示例为 http 服务，https 服务配置方法相同， vhost_http_port 替换为 vhost_https_port， type 设置为 https 即可。

1.修改 frps.ini 文件，配置一个名为 web 的 http 反向代理，设置 http 访问端口为 8080，绑定自定义域名www.yourdomain.com:

```
# frps.ini
[common]
bind_port = 7000
vhost_http_port = 8080

[web]
type = http
custom_domains = www.yourdomain.com
auth_token = 123
```



2.启动 frps

```
./frps -c ./frps.ini
```



![](https://ww1.sinaimg.cn/large/007i4MEmgy1g0wyw7nik9j31g40kutgn.jpg)

3.修改 frpc.ini 文件，设置 frps 所在的服务器的 IP 为 x.x.x.x，local_port 为本地机器上 web 服务对应的端口：

```
# frpc.ini
[common]
server_addr = x.x.x.x
server_port = 7000
auth_token = 123

[web]
type = http
local_port = 80
```



4.启动 frpc：

```
./frpc -c ./frpc.ini
```



5.将 www.yourdomain.com 的域名 A 记录解析到 IP x.x.x.x，如果服务器已经有对应的域名，也可以将 CNAME 记录解析到服务器原先的域名

6.通过浏览器访问 [http://www.yourdomain.com:8080](http://www.yourdomain.com:8080/) 即可访问到处于内网机器上的 web 服务。

![](https://ww1.sinaimg.cn/large/007i4MEmgy1g0wywn35y4j31160m843z.jpg)

这里我就以内网web服务器为例 其他的服务 也可以 自己去看文档吧

我树莓派上Apache已经可以在公网访问了 来看下我这些天的功劳

![](https://ww1.sinaimg.cn/large/007i4MEmgy1g0wywzacf1j30ng15eabv.jpg)![](https://ww1.sinaimg.cn/large/007i4MEmgy1g0wyxujfw5j30ni15e0uc.jpg)