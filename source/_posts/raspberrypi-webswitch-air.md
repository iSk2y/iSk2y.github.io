---
title: raspberrypi-web远程控制空调
date: 2017-05-04 23:52:14
tags: 
 - raspberrypi
categories: raspberrypi
---

基于树莓派-web远程控制空调，有点智能家居的味道 ，既然答辩已经结束 那就写篇文章记录下

但是，由于其他原因 。源码不能直接分享各位小伙伴，不好意思了，如果有意，可以来找我交流下
<!-- more -->
![](https://ww1.sinaimg.cn/large/007i4MEmgy1g0wz68cspvj31h60yqqdv.jpg)
![](https://ww1.sinaimg.cn/large/007i4MEmgy1g0wz7wlelxj30ni15e0uc.jpg)

上面两幅图分别是 树莓派 和web页面 （代码纯原创 ，估计这么渣的代码也找不到第二个了）

![](https://ww1.sinaimg.cn/large/007i4MEmgy1g0wz8ffsk0j30xo0l2my5.jpg)

我流程图画成这样 应该不用过多解释了吧

接下来我们来看看

## 红外遥控的原理

> 1.红外遥控原理 红外遥控器是利用一个红外发射管，以红外光为载体来将按键信息传递给接收端的设备。红外遥控系统主要分为调制、发射 和接收三部分

**调制：**为了免受其他信号的干扰 通常会调制在特定的载波频率上 ，在此发送红外信号的二进制脉冲码

**发射：**发射的是一串二进制的脉冲码 如何分辨 0 和 1 呢 这就是PWM脉冲宽度编码 根据脉冲宽度来区别

![](https://ww1.sinaimg.cn/large/007i4MEmgy1g0wz8v4zvyj30zq0j8gmd.jpg)

这里我说明下 并不是所有脉冲编码都是一样的 这次我控制的美的空调中的就是这样

0.54ms低电平 0.54高电平代表 0

0.54ms低电平 1.62ms高电平代表1

然后紧接着来看下空调的编码

![img](https://isky0t.github.io/images/raspberrypi08.png)

经过我自己测试，然后翻阅了资料发现我这次控制的空调 编码如下，当然 还有定时开、关机 更复杂的编码 我没有去管他。。这个主要就是这样，我感觉够用了，远程开关空调，我感觉定时功能 就是多余的了，你说呢？？？

![](https://ww1.sinaimg.cn/large/007i4MEmgy1g0wz9mp9i8j317006kgm6.jpg)

紧接着 把0和1替换成 波形

![](https://ww1.sinaimg.cn/large/007i4MEmgy1g0wz9zz8xuj31rg0ie4a4.jpg)

**注意下 最后还要加一个结束码 540 就是0.54ms**

然后利用，红外数据发送的程序，将他发送出去 （由于我硬件太白痴 所以我使用了lirc）

lirc 的灵活使用 将配置文件中的 KEY 改成我们的raw码，（这里要说下，KEY取的名字，并不局限于namespace中的，我自己试了，其实都可以取！！） 具体怎么使用lirc，百度下吧，教程很多，不过很多都是重复copy的！！这里要说下有个特别要注意的地方，raw码中间的空格要替换成5个，否则配置文件就会显示错误！！

最后可以使用

```
irsend SEND_ONCE /home/pi/lircd.conf KEY_POWER
```



ircd.conf是配置命名 KEY_POWER是键名 自己灵活运用

*这里要说明下 为啥空调只能通过发送raw码的方式 因为空调是状态码！！*

## web原理

1.web服务器搭建在树莓派 上 ，然后运用上个帖子介绍的frp内网穿透的办法 将web网站 开放到外网上 ，同时使用里了里面的 一个认证访问的设置 输入用户名和密码 才可以访问到反向代理的页面

2.这边我并没有用到MySQL数据库 直接利用的php脚本去执行的树莓派的终端命令 令它运行发送红外数据的命令,一开始我还以为速度会有点慢 没想到几乎没有差别