---
title: H5唤起APP进行分享的尝试
date: 2019-01-04 16:11:05
tags:
 - JavaScript
 - font-end
 - xiuno BBS
 - 编程开发
categories: font-end
---


最近很久没有写blog和note，倒是过家家的开发日志简单草草写了一点。这次记录下这个学习过程

<!-- more -->

# H5唤起APP进行分享







## 由来

我们的 ["通达有你"](https://bbs.tdsast.cn/)，web h5页面的分享功能体验太差了，我一直想改变提高体验度。

通常点分享然后跳转到另一个页面，比如QQ、空间、微博，还有微信。微信通常要扫二维码分享，（我们只有一个手机啊，还要再屏幕上扫二维码，通常要是我是尝试分享者，微信这么麻烦的分享我肯定是不会继续分享了）

所以刚有空我就想试试更好的方法



**疑问？**

然后我平时留意几大互联网的巨头的h5页面，他也是可以进行APP唤起的，这到底是怎么做到的？

不过这种情况分为两种：

1. 唤起自己产品的APP

2. 唤起第三方APP

   ![](https://ww1.sinaimg.cn/large/007i4MEmgy1fyujg89vpfj30as0itmyz.jpg)

![](https://ww1.sinaimg.cn/large/007i4MEmgy1fyujnlnhh4j30ay0iy408.jpg)

而我的目的是第二种，我们是要唤起第三方APP进行分享



## 分析实现方式

经过我 面向搜索引擎 的一顿操作，了解到一些资料和方式，其中要实现在h5唤起APP的主要采用

1. URL Scheme | Intent | Universal Link（也称：深度链接）
2. 通过浏览器的内置native分享接口



### URLScheme



#### URL Scheme 是什么

我们来看一下 URL 的组成：

```
[scheme:][//authority][path][?query][#fragment]
```

我们拿 `https://www.baidu.com` 来举例，scheme 自然就是 `https` 了。

就像给服务器资源分配一个 URL，以便我们去访问它一样，我们同样也可以给手机APP分配一个特殊格式的 URL，用来访问这个APP或者这个APP中的某个功能(来实现通信)。APP得有一个标识，好让我们可以定位到它，它就是 URL 的 Scheme 部分。



#### 常用APP的 URL Scheme

| APP        | 微信      | 支付宝    | 淘宝      | 微博         | QQ     | 知乎     | 短信   |
| ---------- | --------- | --------- | --------- | ------------ | ------ | -------- | ------ |
| URL Scheme | weixin:// | alipay:// | taobao:// | sinaweibo:// | mqq:// | zhihu:// | sms:// |



#### URL Scheme 语法

上面表格中都是最简单的用于打开 APP 的 URL Scheme，下面才是我们常用的 URL Scheme 格式：

```
     行为(应用的某个功能)    
            |
scheme://[path][?query]
   |               |
应用标识       功能需要的参数
```



#### 搜集到的常用Scheme

| 应用名称          | URL Scheme                                                   |
| ----------------- | ------------------------------------------------------------ |
| 微博              | weibo://                                                     |
| QQ                | mqq://                                                       |
| QQ群组            | mqqapi://card/show_pslcard?src_type=internal&version=1&card_type=group&uin={QQ群号} |
| QQ联系人          | mqqapi://card/show_pslcard?src_type=internal&version=1&uin={QQ号码} |
| 支付宝            | alipay://                                                    |
| 微信              | weixin://                                                    |
| 微信              | wechat://                                                    |
| 微信-扫一扫       | weixin://dl/scan                                             |
| 微信-反馈         | weixin://dl/feedback                                         |
| 微信-朋友圈       | weixin://dl/moments                                          |
| 微信-设置         | weixin://dl/settings                                         |
| 微信-消息通知设置 | weixin://dl/notifications                                    |
| 微信-聊天设置     | weixin://dl/chat                                             |
| 微信-通用设置     | weixin://dl/general                                          |
| 微信-公众号       | weixin://dl/officialaccounts                                 |
| 微信-游戏         | weixin://dl/games                                            |
| 微信-帮助         | weixin://dl/help                                             |
| 微信-反馈         | weixin://dl/feedback                                         |
| 微信-个人信息     | weixin://dl/profile                                          |
| 微信-功能插件     | weixin://dl/features                                         |
| 虾米音乐          | xiami://                                                     |
| chrome            | googlechrome://                                              |
| 微博国际版        | weibointernational://                                        |
| 摩拜单车          | mobike://                                                    |
| ofo               | ofoapp://                                                    |
| 有道云笔记        | youdaonote://                                                |
| 印象笔记          | evernote://                                                  |
| 今日头条          | snssdk141://                                                 |
| 网易新闻          | newsapp://                                                   |
| 网易云音乐        | orpheuswidget://                                             |
| QQ音乐            | qqmusic://                                                   |
| QQ音乐最近播放    | qqmusic://today?mid=31&k1=2&k4=0                             |
| 美团外卖          | meituanwaimai://                                             |
| 美团              | imeituan://                                                  |
| Gmail             | googlegmail://                                               |
| 网易邮箱          | neteasemail://                                               |
| QQ邮箱            | qqmail://                                                    |
| 腾讯视频          | tenvideo://                                                  |
| 爱奇艺            | iqiyi://                                                     |
| 12306             | cn.12306://                                                  |
| 有道词典          | yddict://                                                    |
| 钉钉              | dingtalk://                                                  |



#### 我进入深坑

看到上面这些资料的我，思考了**很久**如何去找具体的path和query部分。如果能找到这两块的详细参数，那么我们就可以在APP间相互调用和传参了，就和普通URL一样了



但是，我寻找了好久，或许是我不懂这里面的方式吧，或许是我也不太懂Android方面的知识，就是没人在网上提及到具体的path和query，



后来想想，也是挺正常的，大多网站和企业只是想唤起自己的APP，那如何使用这些URL Scheme呢



#### 大坑

先预告一下还有比较大的坑就是，scheme的触发方式会被浏览器拦截，如果是QQ和微信自带的内置浏览器，使用Scheme是无效的，除非是自家人或者在它的白名单内（比如几大巨头），这就导致了如果你在QQ和微信内点击网页内的分享，不好意思，啥用没有。。。不过可以做个遮罩层用他们内置浏览器的右上角分享功能。

- 微信、微博、手百、QQ浏览器等。

  这些应用能阻止唤端是因为它们直接屏蔽掉了 URL Scheme 。接下来可能就有看官疑惑了，微信中是可以打开大众点评的呀，微博里面可以打开优酷呀，那是如何实现的呢？

  它们都各自维护着一个白名单，如果你的域名在白名单内，那这个域名下所有的页面发起的 URL Scheme 就都会被允许。就像微信，如果你是腾讯的“家属”，你就可以加入白名单了，微信的白名单一般只包含着“家属”，除此外很难申请到白名单资质。但是微博之类的都是可以联系他们的渠道童鞋进行申请的，只是条件各不相同，比如微博的就是在你的 APP 中添加打开微博的入口，三个月内唤起超过 100w 次，就可以加入白名单了。

- 腾讯应用宝直接打开 APP 的某个功能

  刚刚我们说到，如果你不是微信的家属，那你是很难进入白名单的，所以在安卓中我们一般都是直接打开腾讯应用宝，ios 中 直接打开 App Store。点击腾讯应用宝中的“打开”按钮，可以直接唤起我们的 APP，但是无法打开 APP 中的某个功能（就是无法打开指定页面）。

  腾讯应用宝对外开放了一个叫做 APP Link 的申请，只要你申请了 APP Link，就可以通过在打开应用宝的时候在应用宝地址后面添加上 `&android_schema={your_scheme}` ，来打开指定的页面了。



感觉腾讯应用宝是个做文章的很地方，没准之后会尝试下



### 浏览器的内置分享原生分享

浏览器自带有分享到微信和QQ的功能，但不是每个都提供接口来供网页调用。即使有提供，浏览器暴露的api不一样，各家有各家的规则和方式。

但好在网上找到了两个前辈造的轮子[NativeShare](https://github.com/fa-ge/NativeShare)和[uc-qq-share-to-wechat](https://github.com/AngusFu/uc-qq-share-to-wechat) ，也都是调用各个浏览器的原生分享功能，但前者里面我还看到了部分Scheme的方式。



主要的浏览器有：

- QQ浏览器
- UC浏览器
- 微信自带浏览器
- QQ自带浏览器
- QQ空间APP
- 百度浏览器
- 百度APP自带浏览器
- ios 搜狗浏览器
- Safari浏览器
- 其他





## 触发方式

现在来说说触发方式，大致上有三种：

### iframe方案

```html
<iframe src="sinaweibo://qrcode">
```

在未安装 app 的情况下，不会去跳转错误页面。但是 iframe 在各个系统以及各个应用中的兼容问题还是挺多的，我自己并没有使用这种。

```javascript
var last = Date.now(),
	doc = window.document,
	ifr = doc.createElement('iframe');

//创建一个隐藏的iframe
ifr.src = nativeUrl;
ifr.style.cssText = 'display:none;border:0;width:0;height:0;';
doc.body.appendChild(ifr);

setTimeout(function() {
	doc.body.removeChild(ifr);
	//setTimeout回小于2000一般为唤起失败 
	if (Date.now() - last < 2000) {
    	if (typeof onFail == 'function') {
        	onFail();
    	} else {
        	//弹窗提示或下载处理等
    	}
	} else {
    	if (typeof onSuccess == 'function') {
        	onSuccess();
    	}
	}
}, 1000);
```

> iframe方案的唤起原理是: 程序切换到后台时，计时器会被推迟(计时器不准的又一种情况)。如果app被唤醒那么网页必然就进入了后台，如果用户从app切回来，那么时间一般会超过2s;若app没有被唤起，那么网页不会进入后台，setTimeout基本准时触发，那么时间不会超过2s。



### a标签唤起

```html
<a href="mqqapi://card/show_pslcard?src_type=internal&version=1&uin=123456">QQ临时交流</a>
```

上面这个a标签你可以自己尝试一下，是可以直接唤起的，a标签如果目标scheme错误，即应用不存在也不会报错。



### location.href跳转

```javascript
window.location.href = 'sinaweibo://qrcode';
```

我自己大多使用click的方式配合location来进行跳转，自己没有在各大平台和各个浏览器版本上做太多实验，在一个博文中看到

> URL Scheme 在 ios 9+ 上诸如 safari、UC、QQ浏览器中， iframe 均无法成功唤起 APP，只能通过 window.location 才能成功唤端。



某篇博文中对三种唤起方式进行了测试

```
。X表示唤起失败，√表示唤起成功
。红色标记表示进入页面直接唤起，绿色表示人工事件操作后唤起
。ios测试机:iphone 6p;android测试机:小米1s
```

**iframe唤起app测试结果**

![图片显示](https://ww1.sinaimg.cn/large/007i4MEmgy1fyumi02b2uj30bu05bgly.jpg)

**window.location.href唤起app测试结果**

![图片显示](https://ww1.sinaimg.cn/large/007i4MEmgy1fyumigib7qj30bp05a0t3.jpg)

**a标签唤起app测试结果**

![图片显示](https://ww1.sinaimg.cn/large/007i4MEmgy1fyumj0byirj30bq057wes.jpg)

**iframe和window.location.href唤起对比**

![图片显示](https://ww1.sinaimg.cn/large/007i4MEmgy1fyumjhaketj30fv05974t.jpg)

**iframe、window.location.href和a标签唤起三者对比**

![图片显示](https://ww1.sinaimg.cn/large/007i4MEmgy1fyumjqhqdvj30fy05b0tb.jpg)



对比iframe唤起和location.href，我们可以发现：

1. 对于ios来说，location.href跳转更合适，因为这种方式可以在Safari中成功唤起app。Safari作为iphone默认浏览器其重要性就不用多说了，而对于微信和qq客户端，ios中这两种方式都没有什么卵用==
2. 对于Android来说，在进入页面直接唤起的情况下，iframe和location.href是一样的，但是如果是事件驱动的唤起，iframe唤起的表现比location.href要更好一点。
3. 通过测试可以发现，进入页面直接唤起和事件驱动的唤起，对于很多浏览器，两者的表现是不同的，简单来说，直接唤起的失败更多。

> 以上测试可能随时间已经有出入和变化，仅供参考



## 选择

看了这些资料后，开始动手对网站的分享功能再次二开，目前都是采用a标签进行一个简单的跳转，去对应社交平台的分享url的api接口提交各个参数。

![](https://ww1.sinaimg.cn/large/007i4MEmgy1fyukw1uiyij30y4073gno.jpg)



### 尝试

准备先引入大佬的NativeShare轮子，因为我在他的在线demo尝试了，我手机自带浏览器、模拟器的自带浏览器、模拟器的QQ浏览器，都可以很好的分享出来，于是首先引入他的方式。



### 思路

```javascript
$(document).ready(function () {
    $(".social-share-icon ").on("click",function (e) {
        e.preventDefault();
        shareHref = $(this).attr("href");

        if ($(this).hasClass("icon-qq")){

            call('qqFriend',shareHref);

        }

    })
})
```



对目前的各个a标签，在dom加载完成时添加一个click事件，阻止a标签的触发，然后把href地址记录下来。然后再去触发相应的NativeShare中的call方法。



但是我发现在原生的浏览器里面竟然没触发。。那就做个降级处理吧。

```javascript
function call(command) {
            try {
                nativeShare.call(command)
            } catch (err) {
                // 在这里写降级处理的内容
                alert(err.message)
            }
        }
```

在catch到抛出到异常后，自己在用原生的URL scheme 尝试触发，如果还是不能触发，那就没办法咯~~

```javascript
var url_scheme = '//share/to_fri?src_type=web&version=1&file_type=news&title=' + Base64.encode(shareData.title) + '&thirdAppDisplayName=5o6M5LiK55CG5bel5aSn&url=' + Base64.encode(shareData.link) + '&description=' + Base64.encode(shareData.desc);
            location.assign('mqqapi:' + url_scheme)
            setTimeout(function () {
                location.assign('timapi:' + url_scheme)
            }, 2000)
```

这个是我在网上找到的一个qq 分享的URL scheme，记得要base64处理下各个内容，还有个缺陷就是没有icon了，图片显示不了了。



### 细节处理

刚刚都是说手机移动端访问，如果要是电脑端访问的话，还是触发这个肯定不成功。所以在最开始要先判断下ua。根据ua来做出下一步动作。简单的放下ua判断代码

```javascript
var browser = {
    versions: function() {
        var u = navigator.userAgent, app = navigator.appVersion;
        return {     //移动终端浏览器版本信息
            trident: u.indexOf('Trident') > -1, //IE内核
            presto: u.indexOf('Presto') > -1, //opera内核
            webKit: u.indexOf('AppleWebKit') > -1, //苹果、谷歌内核
            gecko: u.indexOf('Gecko') > -1 && u.indexOf('KHTML') == -1, //火狐内核
            mobile: !!u.match(/AppleWebKit.*Mobile.*/), //是否为移动终端
            ios: !!u.match(/\(i[^;]+;( U;)? CPU.+Mac OS X/), //ios终端
            android: u.indexOf('Android') > -1 || u.indexOf('Linux') > -1, //android终端或uc浏览器
            iPhone: u.indexOf('iPhone') > -1, //是否为iPhone或者QQHD浏览器
            iPad: u.indexOf('iPad') > -1, //是否iPad
            webApp: u.indexOf('Safari') == -1, //是否web应该程序，没有头部与底部
            qq: u.indexOf('MQQBrowser') > -1,//QQ浏览器
            uc: u.indexOf('UCBrowser') > -1// UC浏览器

        };
    } (),
    language: (navigator.browserLanguage || navigator.language).toLowerCase()
}
```



```javascript
// 先判断是否是PC，如果是PC端 就直接使用链接来分享
        if (browser.versions.mobile || browser.versions.android || browser.versions.ios){
            // if (browser.versions.qq && command==='qqFriend' || command==='qZone'){
            //     throw new Error;
            // }
            nativeShare.call(command)
        }else{
            // 使用原链接分享
            location.assign(e)
        }
```



### 效果



最后测试了几个浏览器分享效果

|   浏览器   | 华为自带 | X浏览器 | 模拟器自带 | 模拟中QQ浏览器 | APK内置 |
| :--------: | :------: | :-----: | :--------: | :------------: | :-----: |
|   QQ好友   |    N     |    Y    |     Y      |       Y        |    Y    |
|   QQ空间   |    N     |    Y    |     Y      |       Y        |    Y    |
| 微信朋友圈 |    Y     |    N    |     N      |       Y        |    N    |



![](https://ww1.sinaimg.cn/large/007i4MEmgy1fyum7dmcxkj30cg0qrgni.jpg)

![](https://ww1.sinaimg.cn/large/007i4MEmgy1fyum8717wrj30jc0g8q47.jpg)

看来QQ浏览器果然通过原生的内置分享api都能达到分享唤起，但是在其他浏览器就有部分不兼容了。后面有需要在继续调整优化吧，



## Reference

> H5唤起APP指南 https://suanmei.github.io/2018/08/23/h5_call_app/
>
> h5唤起app http://echozq.github.io/echo-blog/2015/11/13/callapp.html

# 兼容是重点