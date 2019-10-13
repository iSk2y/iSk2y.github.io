---
title: xiunoBBS正方插件开发记录
date: 2018-12-18 23:11:00
tags:
 - PHP
 - xiuno BBS
categories: PHP
---

# 正方认证插件



基于XiunoBBS开发的正方认证插件

<!-- more -->

> PS：开发插件请把DEBUG设置为2，否则是不会及时编译文件到tmp目录下的，具体原因看博文

# 插件配置信息

> conf.json

```json
{
  "name":"正方认证",
  "brief":"接入正方认证，认证后添加到学生组",
  "version":"1.0",
  "bbs_version":"4.0.4",
  "installed":1,
  "enable":1,
  "hooks_rank":[],
  "overwrites_rank":[],
  "dependencies":[]
}
```



# 页面添加认证栏

先在my页面左侧栏添加一栏 正方认证 ，作为认证的超链接入口。这里要利用`my_common_my_thread_after.htm`这个hook，具体原因看 `view/htm/my.common.template.htm`文件。

## PC端

1. 新建`hook/my_common_my_thread_after.htm`

2. 写入内容

   ```html
           <a href="<?php echo url('my-zfauth');?>" class="list-group-item list-group-item-action" data-active="menu-my-zfauth"><?php echo lang('zfauth');?></a>
   ```

   ~~正方认证这几个字直接硬编码了，因为考虑到应该用到的语言项内容不多，暂时就不写lang了，后面有需要再说~~

   真香，我还是用了lang（hook文件夹下建立`lang_zh_cn_bbs.php`）

![image.png](https://upload-images.jianshu.io/upload_images/14657587-463a2f35badadb89.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## mobile端

因为是响应式的，但是我也不知为啥代码分开写了，那就还要触发另一个hook

1. 新建`hook/my_common_mobile_my_thread_after.htm`
2. 写入内容

```html
<a role="button" class="btn btn-secondary" data-active="menu-my-zfauth" href="<?php echo url('my-zfauth');?>"><?php echo lang('zfauth');?></a>
```

![image.png](https://upload-images.jianshu.io/upload_images/14657587-7787a6a69d3df64f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> PS：消息 是我另外添加的插件

正方认证的字样已经出现了，但是现在点了肯定是空白的，接下来我们添加路由



# 添加路由

利用`my_end.php`这个hook

1. 新建`hook/my_end.php`
2. 书写内容为

```php
<?php exit;
elseif($action == 'zfauth') {

	if($method == 'GET') {

		$active = 'zfauth'; // bootstrap active状态显示
		
		$header['title'] = lang('zfauth');
		$header['mobile_title'] = lang('zfauth');
		// view视图部分
		include _include(APP_PATH.'plugin/isk2y_zfauth/view/htm/my_zfauth.htm');

	} elseif($method == 'POST') {

	}	
}
?>
```

暂时没有些其他什么逻辑

# 编写view页面

然后包含my_zfauth.htm并\_include

> plugin/isk2y_zfauth/view/htm/my_zfauth.htm

```html
<template include="./plugin/isk2y_zfauth/view/htm/my_zfauth.template.htm">
    <slot name="my_body">
        <?php echo lang('zfauth');?>
    </slot>
</template>

<script>
    $('a[data-active="my-zfauth"]').addClass('active');
</script>

```

其中模板又使用了my_zfauth.template.htm

> plugin/isk2y_zfauth/view/htm/my_zfauth.template.htm

```html
<template include="./view/htm/my.common.template.htm">
    <slot name="my_nav">
        <ul class="nav nav-tabs card-header-tabs">
            <!--{hook my_nav_notice_before.htm}-->
            <li class="nav-item">
                <a class="nav-link" href="<?php echo url("my-zfauth");?>" data-active="my-zfauth"><?php echo lang('zfauth');?></a>
            </li>
            <!--{hook my_nav_notice_after.htm}-->
        </ul>
    </slot>
</template>

<script>
    $('a[data-active="menu-my-zfauth"]').addClass('active');
</script>
```

![image.png](https://upload-images.jianshu.io/upload_images/14657587-b7e9b4218f31e3e3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



然后经过我不堪入目的前端修改下就凑合这样子吧~~

![image.png](https://upload-images.jianshu.io/upload_images/14657587-a66b43e352d7e237.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![1544862193207](C:\Users\isk2y\AppData\Roaming\Typora\typora-user-images\1544862193207.png)

修改后的源码

> plugin/isk2y_zfauth/view/htm/my_zfauth.template.htm

```html
<template include="./plugin/isk2y_zfauth/view/htm/my_zfauth.template.htm">
    <slot name="my_body">
        <div>
            <form id="zfform" action="<?php echo url('my-zfauth');?>">
                <div class="form-group row">
                    <label for="zfusername" class="col-3 col-form-label text-right"><?php echo lang('zfusername');?>:</label>
                    <div class="col-9 col-sm-5 col-xl-4">
                        <input type="text" class="form-control" id="zfusername" name="zfusername" placeholder="就是你的学号啦！">
                    </div>
                </div>

                <div class="form-group row">
                    <label for="zfpassword" class="col-3 col-form-label text-right"><?php echo lang('zfpassword');?>:</label>
                    <div class="col-9 col-sm-5 col-xl-4">
                        <input type="password" class="form-control" id="zfpassword" name="zfpassword" placeholder="盖住屏幕不要给别人看密码!">
                    </div>
                </div>
                <div class="form-group row d-flex align-items-center">
                    <label for="zfvaptcha" class="col-3 col-form-label text-right"><?php echo lang('zfverifycode');?>:</label>
                    <div class="col-6 col-sm-3 col-xl-2 ">
                        <input type="text" class="form-control" id="zfvaptcha" name="zfvcode" placeholder="→_→">
                    </div>
                    <div class="col-3 col-sm-2 col-xl-1 d-flex justify-content-center">
                        <img src="<?php echo $vcodesrc;?>" alt="正方验证码拉取失败~~" id="vcode" onclick="getVcode()">
                    </div>
                </div>
                <div class="form-group row col-6 col-sm-5 col-xl-4 offset-3">
                    <button type="button" class="btn btn-primary btn-block" id="submit" data-loading-text="<?php echo lang('zfauthing');?>..."><?php echo lang('zfauth');?></button>
                </div>
            </form>
        </div>
    </slot>
</template>

```



# 编写自己的func内容

上面的初步工作已经完成，然后就是用PHP去模拟登陆正方了，这边不详细展开如何模拟获取信息的过程了，不属于本文重点内容，不过要吐槽的是 正方真的有点烦！！！！还有总结的几个注意点

1. 模拟登陆正方最重要的是 cookie，访问default.aspx和验证码都会检查你的cookie是否存在如果没有就set-cookie给你，所以你得在第一次访问任何一个页面的时候把cookie存下来，下次继续用这个cookie去访问其他页面，这是为了让正方认得你，因为cookie中带有session的辨认符。至于怎么放我思考了两个点
   - 把cookie作为临时变量，但是你得确保在一次访问脚本中就登录，显然这是不可能的，除非你已经知道用户的密码了，这个当然可以咯
   - 把cookie存放在session里，持久化储存，肯定可以的。但是会消耗session资源而且可能在并发大的时候造成session阻塞。
   - 把cookie设置在你本地，随用随取，但是污染了本地cookie，其实谈不上污染就多一个值嘛~~
2. 正方登陆页面的两个hidden隐藏表单内容，得在先前取到，至于放在哪里？
   - 放在session里或者其他持久化储存中
   - 放在前端！！把从正方拿到的表单给用户传过去放在前端，这样他下次提交的时候就能拿到了，嘻嘻~~
3. 登陆的302 要跟进，curl别忘记设置。
4. 如果出现访问个人信息页面xsgrxx 出现object move to here，那肯定是你cookie不对了，自己debug下。
5. 记得设置好编码，正方不用utf-8的！！
6. 可以在curl中设置proxy，提高debug效率！首选fiddler。



好了，总结正方的内容，下次我肯定不会用PHP写爬虫了！！！打死我也不！！回到Python，一把梭就完了

```php
<?php

include _include(APP_PATH.'plugin/isk2y_zfauth/utils/simple_html_dom.class.php');

/**
 * 获取正方登陆表单中必要的form元素，并获取到cookie
 * @param $url 正方url
 * @return array
 */
function getZfinit($url){
    $ret = array();

    $headerArr = array(
        'Referer:' . $url,
    );
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, $url);
    curl_setopt($ch, CURLOPT_HEADER, TRUE);    //启用时会将头文件的信息作为数据流输出,为了取到cookie
    curl_setopt($ch, CURLOPT_NOBODY, FALSE);   //表示需要response body
    curl_setopt($ch, CURLOPT_HTTPHEADER , $headerArr );  //构造header头部
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, TRUE); // 将获取到的内容返回，而不是直接输出
//    curl_setopt($ch,CURLOPT_PROXY,'127.0.0.1:8888');//设置代理服务器 fiddler debug用的
    curl_setopt($ch, CURLOPT_FOLLOWLOCATION, FALSE); //表示不重定向
    curl_setopt($ch, CURLOPT_TIMEOUT, 300);
    $result = curl_exec($ch);

    // 返回状态码200 表示正常访问正方成功了
    if (curl_getinfo($ch, CURLINFO_HTTP_CODE) == '200') {

        $view = array();
        preg_match_all('/<input type="hidden" name="__VIEWSTATE" value="([^<>]+)" \/>/', $result, $view['state']); //获取__VIEWSTATE字段并存到$view数组中
        preg_match_all('/<input type="hidden" name="__VIEWSTATEGENERATOR" value="([^<>]+)" \/>/', $result, $view['stategenerator']);

        // 匹配cookie
        if(preg_match('/Set-Cookie:[\s+](.*);/iU',$result,$cookie)){
            // 判断下当前有没有获取过cookie了 ,如果获取过就别在set了

            header('Set-Cookie: '.$cookie[1].'; path=/');
            $ret['status'] = true;
            // 想来想去 这个view表单hidden项还是存在session中吧，因为后面还要用到，或者把表单项显示到前端去
            if(session_status() === 1){
                session_start();
            }elseif(session_status() === 0){
                exit('会话是被禁用的。请打开session支持');
            }
            $_SESSION['zfview'] = $view;
            session_write_close();

            $ret['cookie'] = $cookie[1];
        }else{
            $ret['status'] = false;
            $ret['reason'] = 'match_cookie_failed';
        }
    }else{
        $ret['status'] = false;
        $ret['reason'] = 'open_zf_failed';

    }
    curl_close($ch);
    return $ret;

}

/**
 * import！ 确保这个函数在getZfinit后调用，否则cookie取不到相同登陆会失败
 * @param $codeUrl 验证码的url
 * @param $cookie
 * @param $vcodepath 存放验证码的地方
 * @return bool
 */
function getZfVCode($codeUrl, $cookie, $vcodepath){
    $getstatus = true;
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, $codeUrl);
    curl_setopt($ch, CURLOPT_COOKIE, $cookie);  //带cookie访问
//    curl_setopt($ch,CURLOPT_PROXY,'127.0.0.1:8888');//设置代理服务器 fiddler debug用的
    curl_setopt($ch, CURLOPT_HEADER, TRUE);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, TRUE);
    $img = curl_exec($ch);  //执行curl
    if (curl_getinfo($ch,CURLINFO_HTTP_CODE) != '200' ){
        $getstatus = false;
    }else{
        list($head,$body) = explode("\r\n\r\n",$img);
        if(preg_match('/Set-Cookie:[\s+](.*);/iU',$head,$cookie)){
            header('Set-Cookie: '.$cookie[1].'; path=/');
        }
    }
    curl_close($ch); // 要注意关闭curl才能写文件

    // 如果你不想把验证码下载到本地 可以多加一个路由，将图片展示到该路由上即可，想想怎么改？而且这样cookie也不用写入session了
    $fp = fopen($vcodepath."verifyCode.jpg","w");  //文件名
    fwrite($fp,$body);  //写入文件
    fclose($fp);
    return $getstatus;
}

/**
 * 获取验证码的路由方法，直接echo出来就是图片了，已经设置header
 * @param $codeUrl 正方验证码url
 * @return mixed 拿回验证码的raw数据
 */
function getZfVcodeRouter($codeUrl){

    $headerArr = array(
        'Referer:http://自己写正方地址！！',
    );
    if(isset($_COOKIE['ASP_NET_SessionId'])){
        $headerArr[] = 'Cookie: ASP.NET_SessionId='.$_COOKIE['ASP_NET_SessionId'];
    }
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, $codeUrl);
    curl_setopt($ch, CURLOPT_HEADER, TRUE);    //表示需要response header
    curl_setopt($ch, CURLOPT_NOBODY, FALSE); //表示需要response body
    curl_setopt($ch, CURLOPT_HTTPHEADER , $headerArr );  //构造IP
//    curl_setopt($ch,CURLOPT_PROXY,'127.0.0.1:8888');//设置代理服务器 fiddler debug用的
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, TRUE);
    curl_setopt($ch, CURLOPT_FOLLOWLOCATION, FALSE);
    //curl_setopt($ch, CURLOPT_AUTOREFERER, TRUE);
    curl_setopt($ch, CURLOPT_TIMEOUT, 120);
    $result = curl_exec($ch);
    if (curl_getinfo($ch,CURLINFO_HTTP_CODE) != '200' ){
        $getstatus = false;
    }else{
        list($head,$body) = explode("\r\n\r\n",$result);
        if(preg_match('/Set-Cookie:[\s+](.*);/iU',$head,$cookie)){
            header('Set-Cookie: '.$cookie[1].'; path=/');
        }
        header('Content-Type:image/Gif');
        $result = $body;
    }
    curl_close($ch); // 要注意关闭curl才能写文件
    return $result;

}

/**
 * @param $url 正方url
 * @param $xh 学号
 * @param $pw 密码
 * @param $code 验证码
 * @return array('status'=> bool, 'data'=> string)
 */
function zfLoginPost($url, $xh, $pw, $code) {
    $ret = array(
        'status'=>false,
    );

    if(session_status() === 1){
        session_start();
    }elseif(session_status() === 0){
        exit('会话是被禁用的。请打开session支持');
    }


    $post=array(
        '__VIEWSTATE'=>$_SESSION['zfview']['state'][1][0],
        '__VIEWSTATEGENERATOR' =>$_SESSION['zfview']['stategenerator'][1][0],
        'txtUserName'=>$xh,
        'Textbox1'=>'',
        'TextBox2'=>$pw,
        'txtSecretCode'=>$code,
        'RadioButtonList1'=>iconv('utf-8','gb2312','学生'),  //“学生”的gbk编码
        'Button1'=>'',
        'lbLanguage'=>'',
        'hidPdrs'=>'',
        'hidsc'=>''
    );
    session_write_close();

    $post = http_build_query($post);

    $headerArr = array(
        'Referer: http://自己写正方地址/default2.aspx',
        'Content-Type: application/x-www-form-urlencoded',
        'Host: 自己写正方地址',
        'Origin: http://自己写正方地址',
        'User-Agent: Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/49.0.2623.87 Safari/537.36',
    );
    if(isset($_COOKIE['ASP_NET_SessionId'])){
        $headerArr[] = 'Cookie: ASP.NET_SessionId='.$_COOKIE['ASP_NET_SessionId'];
    }

    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, $url);
    curl_setopt($ch, CURLOPT_HEADER, 0);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1); //不自动输出数据，要echo才行
    curl_setopt($ch, CURLOPT_FOLLOWLOCATION, 1); //重要，抓取跳转后数据
//    curl_setopt($ch,CURLOPT_PROXY,'127.0.0.1:8888');//设置代理服务器 fiddler debug用的
    curl_setopt($ch, CURLOPT_HTTPHEADER , $headerArr );  //构造header
    curl_setopt($ch, CURLOPT_REFERER, $url); //重要，ASP的302跳转需要referer，可以在Request Headers找到
    curl_setopt($ch, CURLOPT_POSTFIELDS, $post); //post提交数据
    $result = curl_exec($ch);
    // 登陆成功后302跳转会到主页，把这个主页记录下
    if(preg_match('/.*?<a.+"(xsgrxx.aspx\?.*?)".*?>.*?<\/a>.*?/i',$result,$stuInfoUrl)) {
        session_start();
        $_SESSION['zf_profileUrl'] = 'http://自己写正方地址/'.iconv('gb2312','utf-8//ignore',$stuInfoUrl[1]);
        session_write_close();
        $ret['status'] = TRUE;
        $ret['data'] = $result;

    }elseif (preg_match('/alert\(\'(.*?)\'\)/',$result,$reason)){
        // 密码错误 或者 验证码错误 会匹配到的
        $ret['data'] = mb_convert_encoding($reason[1],'UTF-8','GB2312');
    }else{
        $ret['data'] = "open_zf_failed";
    }

    curl_close($ch);
    return $ret;
}

/**
 * @param $url 正方url
 * @param $xh 学号
 * @param $pw 密码
 * @param $code 验证码
 * @return array
 */
function getZfInfo($url, $xh, $pw, $code){
    $ret = array(
        'status'=>false,
        'data'=>''
    );

    $logStatus = zfLoginPost($url, $xh, $pw, $code);
    if (!$logStatus['status']){
//        print_r($logStatus);
//        exit;
        $logStatus['data'] === 'open_zf_failed' AND xn_message(1,lang('open_zf_failed'));
        // 前端input按照错误弹出alert 是根据code构造的选择器，看my_zfauth.htm的js代码，
        $r = mb_strpos($logStatus['data'],'验证码');
        // 错误在logStatus里匹配返回值了 不写在lang里面了
        $r === false AND xn_message('zfpassword',$logStatus['data']);
        xn_message('zfvcode',$logStatus['data']);
    }
    $headerArr = array(
        'Referer: http://自己写正方地址/xs_main.aspx?',
        'User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.110 Safari/537.36',
    );
    if(isset($_COOKIE['ASP_NET_SessionId'])){
        $headerArr[] = 'Cookie: ASP.NET_SessionId='.$_COOKIE['ASP_NET_SessionId'];
    }

    if(session_status() === 1){
        session_start();
    }elseif(session_status() === 0){
        exit('会话是被禁用的。请打开session支持');
    }

    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, $_SESSION['zf_profileUrl']);
    curl_setopt($ch, CURLOPT_HEADER, false);    //表示需要response header
    curl_setopt($ch, CURLOPT_NOBODY, FALSE); //表示需要response body
//    curl_setopt($ch,CURLOPT_PROXY,'127.0.0.1:8888');//设置代理服务器 fiddler debug用的
    curl_setopt($ch, CURLOPT_HTTPHEADER , $headerArr );  //构造header
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, TRUE);
    curl_setopt($ch, CURLOPT_FOLLOWLOCATION, FALSE);
    //curl_setopt($ch, CURLOPT_AUTOREFERER, TRUE);
    curl_setopt($ch, CURLOPT_TIMEOUT, 300);
    $result = curl_exec($ch);

    if(curl_getinfo($ch, CURLINFO_HTTP_CODE) != 200) {
        // 正方打开失败
        $ret['data'] = 'open_zf_failed';
        curl_close($ch);
        return $ret;
    }
//    exit($result);
    /* 删除空白字符 */
    $result=preg_replace("/[\t\n\r]+/","",$result);
    if(preg_match('/.*?(<table class="formlist" width="100%" align="center">.*?<\/table>).*?/U',$result,$stuInfoTable)){ //寻找必要信息
        if($userInfo = parseZfInfo(iconv('gb2312','utf-8',$stuInfoTable[1]))){
            $ret['status'] = true;
            $ret['data'] = $userInfo;// 解析回来的信息

        }else{
            // 个人数据解析失败
            $ret['data'] = 'info_not_found';
        }
    }else{ // 没找到必要信息
        //没找到必要信息
        $ret['data'] = 'info_not_found';
    }
    // 退出下正方吧
    zfLogout();
    curl_close($ch);

    return $ret;
}


/**
 * 解析个人页面信息
 * @param $html_raw 解析的源码，默认给定是xsgrxx页面
 * @return array|bool 是否解析成功
 */
function parseZfInfo($html_raw){

    $html = new simple_html_dom();

    // 学生个人信息页面
    $xsgrxx = array(
        'gender' => '#lbl_xb', //性别
        'stuid' => '#xh', //学号
        'realname' => '#xm', //真实姓名
        'birthday' => '#lbl_csrq', // 生日
//        'uNation' => '#lbl_mz', //民族
        'major' => '#lbl_zymc', //专业
        'academy' => '#lbl_xy', // 学院
//        'uIdnumber' => '#lbl_sfzh', //身份证号 敏感信息！！不敢存啊
    );

    //论文维护信息页面
    $lw_xsxx = array(
        'uStuId' => '#xh', // 学号
        'uRealname' => '#xm', // 真实姓名
        'uAcademy' => '#xy', // 学院
        'uMajor' => '#zy', // 专业
    );
    // 用户信息
    $userInfo = array();

    // 选择要解析的数据页面，设置dom结构点
    $infoDom = $xsgrxx;
    $html->load($html_raw);

    // 抓取数据，如果有一个为空则表示失败
    foreach($infoDom as $key => $dom_info){
        $tmp_fetch = trim($html->find($dom_info, 0)->plaintext);
        /* 判断是否满足错误条件 */
        if(empty($tmp_fetch))return false;
        $userInfo[$key] = $tmp_fetch;
    }

    /* 清理 */
    $html->clear();
    unset($key, $dom_info);
    return $userInfo;
}

/**
 * 简单退出下，就当可怜正方，也顺便清理下cookie
 */
function zfLogout(){

    $headerArr = array(
        'Referer: http://自己写正方地址/xs_main.aspx?',
        'User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.110 Safari/537.36',
    );

    if(isset($_COOKIE['ASP_NET_SessionId'])){
        $headerArr[] = 'Cookie: ASP.NET_SessionId='.$_COOKIE['ASP_NET_SessionId'];
    }

    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, 'http://自己写正方地址/logout.aspx');
    curl_setopt($ch, CURLOPT_HEADER, 0);    //表示需要response header
    curl_setopt($ch, CURLOPT_NOBODY, 0); //表示需要response body
//    curl_setopt($ch,CURLOPT_PROXY,'127.0.0.1:8888');//设置代理服务器 fiddler debug用的
    curl_setopt($ch, CURLOPT_HTTPHEADER , $headerArr );  //构造header
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
    curl_setopt($ch, CURLOPT_FOLLOWLOCATION, 0);
    //curl_setopt($ch, CURLOPT_AUTOREFERER, TRUE);
    curl_setopt($ch, CURLOPT_TIMEOUT, 300);
    $result = curl_exec($ch);
    curl_close($ch);
    setcookie('ASP_NET_SessionId','',time()-86400);
}

function zfauth_create($arr){

    global $time, $longip, $uid;
    $arr['gender'] = $arr['gender']=='男' ? 1 : 0;
    $arr['uid'] = $uid;
    $arr['auth_date'] = $time;
    $arr['auth_ip'] = $longip;
    $r = db_create('zfauth',$arr);
    return $r;
}

?>


```



上面是我配合xiunobbs写的PHP模拟登陆正方，**不具有普遍性**，如果要单用，请替换个别xiunobbs的函数为PHP原函数。

暂时没有写获取课程表啊 成绩之类啊，这里认证不用搞这些。后面这些功能在写就行了，无非就是解析dom

这里有两个getvcode获取验证码的函数说明下，

1. getZfVCode是会把验证码获取下来存放在本地的，然后显示给前端
2. getZfVcodeRouter是一个获取验证码的路由，你把它当路由使就行了，就是src指向，对了别忘了在前端加个random

# 正方扩展表

既然认证成功了就记录下部分信息，后面其他功能中可能会调用，比如自动提醒生日？(#^.^#)

```mysql
### 正方认证信息 ###
DROP TABLE IF EXISTS `bbs_zfauth`;
CREATE TABLE `bbs_zfauth` (
  `uid` int(11) unsigned NOT NULL COMMENT '用户编号',
  `gender` tinyint(1) unsigned NOT NULL DEFAULT '0' COMMENT '性别',
  `stuid` int(10) unsigned NOT NULL COMMENT '学号',
  `realname` char(8) NOT NULL COMMENT '姓名',
  `birthday` int(8) NOT NULL COMMENT '生日',
  `major` char(40) NOT NULL COMMENT '专业',
  `academy` char(16) NOT NULL COMMENT '学院',
  `auth_date` int(11) unsigned NOT NULL DEFAULT '0' COMMENT '认证时间',
  `auth_ip` int(11) unsigned NOT NULL DEFAULT '0' COMMENT '认证ip',
  PRIMARY KEY (`uid`),
  UNIQUE KEY `stuid` (`stuid`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8;
```



这里用uid和bbs的user表做个关联，但是不做外键，在逻辑层实现外间关联就可以了。



# 完善路由controller层

func已经完成，view也部分完成，接下来完成逻辑控制层，没多少代码

```php
<?php exit;
elseif($action == 'zfauth') {

    include _include(APP_PATH.'plugin/isk2y_zfauth/model/zf.func.php');

	if($method == 'GET') {
        $active = 'zfauth';
        $header['title'] = lang('zfauth');
        $header['mobile_title'] = lang('zfauth');
		// 验证码的存放位置
        $vcodesrc = http_url_path().'/tmp/verifyCode.jpg';
        // 判断这个这个路由是不是请求验证码的 命中路由为 ?my-zfauth-vcode.htm 
        $vcode = param(2,'');
        if ( $vcode == 'getcode' and $ajax){
            $getStatus = getZfVCode(lang('zfvcodeurl'), $_COOKIE['ASP_NET_SessionId'],$conf['tmp_path']);

            xn_message($getStatus ? 0 : 1,$getStatus ? $vcodesrc : lang('zfgetvcodeerr'));
        }
        $result = getZfinit(lang('zfloginurl'));
        getZfVCode(lang('zfvcodeurl'), $result['cookie'],$conf['tmp_path']);


		include _include(APP_PATH.'plugin/isk2y_zfauth/view/htm/my_zfauth.htm');

	} elseif($method == 'POST' and $ajax) {
        $zfusername = _POST('zfusername');
        $zfpassword = _POST('zfpassword');
        $zfvcode = _POST('zfvcode');
        $getinfo = getZfInfo(lang('zfloginurl'),_POST('zfusername'),_POST('zfpassword'),_POST('zfvcode'));

        // 如果getinfo状态为失败 就返回给前端错误信息
        !$getinfo['status'] AND xn_message(1,$getinfo['data']);

        $r = zfauth_create($getinfo['data']);

        // 如果获取信息成功，但是插入数据库失败，可能是有其他问题了
        $r===false AND xn_message(1,lang('create_failed'));

        xn_message(0,lang('zfauthsucc'));

	}	
}
?>
```

> PS：不要对开头的exit太关注，这个文件最后是被hook替换掉的，这个exit会消失的，那为何放在这里？安全处理

交代下几个函数使用：

- http_url_path()：获取当前的 URL 路径（不包含文件名和参数）
- param()：获取 GET POST COOKIE REQUEST 参数
- xn_message()：根据请求的方式，输出不同格式的内容，并且终止会话。如果为 AJAX 请求，则输出 json 格式数据：{code: $code, message: $message}。如果为普通请求，则输出 $message。

具体请看 ：https://www.kancloud.cn/xiuno/xiunophp



## 流程&逻辑

这个逻辑controller的逻辑简单就是：

先判断进来的请求路由是否为zfauth，然后判断请求方式，

- 如果是GET，再判断是否带vcode参数，
  - 带vcode参数，就是点击验证码刷新的动作
  - 不带vcode参数，就是访问my-zfauth这个页面，下面就是初始化正方函数和获取验证码函数
- 如果是POST，就是ajax提交的登陆请求，是在登陆正方了，执行获取信息函数，并插入数据库，中途登陆正方的错误信息都在func中我也做了判断并直接返回给前端了。

# 验证码刷新两个函数的实现

## 本地存放形式



这里先用第一种显示验证码的方式，本地先存放，对应的前端请求代码为

```javascript
<script>
    $('a[data-active="my-zfauth"]').addClass('active');
    function getVcode() {
        $.xget("<?php echo url('my-zfauth-getvcode');?>", function(code, message) {
            if(code == 0) {
                $('#vcode').attr('src',message+'?'+Math.random());
            } else {
                alert('错误：'+message);
            }
        })
    }
    var jform = $('#zfform');
    var jsubmit = $('#submit');
    jsubmit.on('click', function() {
        jform.reset();
        jsubmit.button('loading');
        var postdata = jform.serialize();
        $.xpost(jform.attr('action'), postdata, function(code, message) {
            if(code == 0) {
                jsubmit.button(message).delay(1000).location("<?php echo url('my-zfauth');?>");
            } else if(xn.is_number(code)) {
                alert(message);
                jsubmit.button('reset');
            } else {
                jform.find('[name="'+code+'"]').alert(message).focus();
                jsubmit.button('reset');
            }
        });
        return false;
    });

</script>
```

这样也能正常实现点击验证码会刷新

![vcode.gif](https://upload-images.jianshu.io/upload_images/14657587-0fb42f1ef1d87c0f.gif?imageMogr2/auto-orient/strip)



## 直接路由访问

这种方式代码量特别少，但是我因为 莫名其妙 response头部多一个空行 debug了4个小时，后来才知道 世界上最好的语言 尾部?>闭合后不要有空行，否则包含的时候就戏多了，最好纯文本都不要写闭合标签了！！

> plugin/isk2y_zfauth/view/htm/my_zfauth.htm

```html
<img src="<?php echo url('my-zfauth-getcode')?>" alt="正方验证码拉取失败~~" id="vcode" onclick="this.src+='&'+Math.random();">
```

这个view视图只要这样写就行了



逻辑层为

```php
$vcode === 'getcode' AND exit(getZfVcodeRouter(lang('zfvcodeurl')));
```



# 设置校内认证组

```mysql
INSERT INTO bbs_group (`gid`,`name`,`creditsfrom`,`creditsto`,`allowread`,`allowthread`,`allowpost`,`allowattach`,`allowdown`) VALUES ('106','校内认证组','0','0','1','1','1','1','1')
```

既然有认证当然有认证组咯！！~~

这些0和1 分别都是某个权限的真假值。

然后我们在代码里添加逻辑，将认证后的同学加入到这个组别中来

> PS：在板块化权限中，只有对应的组别才能发言和查看等其他功能，所以说认证后的组别会使你有更大权限！！

继续在`hook/my_end.php`中加入这行代码。在上面创建完数据并插入之后添加这个代码，使得当前用户加入校内认证组

```php
        $arr = array('gid'=>'106');
        $r = user_update($uid,$arr);

        $r===false AND xn_message(1,lang('join_group_failed'));
```



最后清理下session中的变量吧

```php
// 清理session中前面设置的数据
        unset($_SESSION['zfview']);
        unset($_SESSION['zf_profileUrl']);
```





# 最终效果

plugin/isk2y_zfauth/hook/my_end.php

```php
<?php exit;
elseif($action == 'zfauth') {

    include _include(APP_PATH.'plugin/isk2y_zfauth/model/zf.func.php');

	if($method == 'GET') {

	    $isauth = db_find_one('zfauth',array('uid'=>$uid));

	    if ($isauth===null){
            $active = 'zfauth';
            $header['title'] = lang('zfauth');
            $header['mobile_title'] = lang('zfauth');


            $vcode = param(2,'');
            /* 验证码存放本地形式的逻辑层代码
             * $vcodesrc = http_url_path().'/tmp/verifyCode.jpg';
             * if ( $vcode == 'getcode' and $ajax){
                $getStatus = getZfVCode(lang('zfvcodeurl'), $_COOKIE['ASP_NET_SessionId'],$conf['tmp_path']);
                xn_message($getStatus ? 0 : 1,$getStatus ? $vcodesrc : lang('zfgetvcodeerr'));
            }*/
            if ( $vcode == 'getcode'){
                $getStatus = getZfVcodeRouter(lang('zfvcodeurl'));
                exit($getStatus);
            }
            //或者可以这样写哦，更简练
//            $vcode === 'getcode' AND exit(getZfVcodeRouter(lang('zfvcodeurl')));
            $result = getZfinit(lang('zfloginurl'));

//            如果是本地验证码形式 这行就要打开！！
//            getZfVCode(lang('zfvcodeurl'), $result['cookie'],$conf['tmp_path']);
        }

		include _include(APP_PATH.'plugin/isk2y_zfauth/view/htm/my_zfauth.htm');

	} elseif($method == 'POST' and $ajax) {
        $zfusername = _POST('zfusername');
        $zfpassword = _POST('zfpassword');
        $zfvcode = _POST('zfvcode');
        $getinfo = getZfInfo(lang('zfloginurl'),_POST('zfusername'),_POST('zfpassword'),_POST('zfvcode'));

        // 如果getinfo状态为失败 就返回给前端错误信息
        !$getinfo['status'] AND xn_message(1, lang($getinfo['data']));

        $r = zfauth_create($getinfo['data']);

        // 如果获取信息成功，但是插入数据库失败，可能是有其他问题了
        $r===false AND xn_message(1,lang('create_failed'));

        $arr = array('gid'=>'106');
        $r = user_update($uid,$arr);

        $r===false AND xn_message(1,lang('join_group_failed'));

        // 清理session中前面设置的数据
        unset($_SESSION['zfview']);
        unset($_SESSION['zf_profileUrl']);

        xn_message(0,lang('zfauthsucc'));

	}	
}
?>
```



plugin/isk2y_zfauth/hook/lang_zh_cn_bbs.php

```
'zfauth'=>'正方认证',
'zfusername'=>'用户名',
'zfpassword'=>'密码',
'zfrole'=>'身份',
'zfrole_student'=>'学生',
'zfrole_teacher'=>'老师',
'zfverifycode'=>'验证码',
'zfloginurl'=>'http://自己写正方地址/default2.aspx',
'zfvcodeurl'=>'http://自己写正方地址/CheckCode.aspx',
'zfgetvcodesucc'=>'获取验证码成功',
'zfgetvcodeerr'=>'获取验证码失败',
'zfauthing'=>'认证中',
'open_zf_failed'=>'正方打开失败',
'info_not_found'=>'信息未找到，你是不是没做班主任评价？',
'zfauthsucc'=>'认证成功',
'join_group_failed'=>'加入校内认证组失败',
```



最后打包最后放GitHub上。。



![image.png](https://upload-images.jianshu.io/upload_images/14657587-f5358f47629406cb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

游客等未认证用户 只有这个版块内容

![image.png](https://upload-images.jianshu.io/upload_images/14657587-79c78734322a3146.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



认证后用户

![image.png](https://upload-images.jianshu.io/upload_images/14657587-e6f5ca69e2e69efc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



# 总结反思

- 关于cookie和session的机制有更多的了解了
- 关于xiunobbs的整个流程机制以及hook和overwrite机制有进一步认识
- 重新预习PHP，dei！世界上最好的语言！真香

由于俺编码coding能力比较弱鸡，还在学习，如果PHP代码写得丑的话，轻喷……

如果有什么问题可以和我交流~~

算是又踏上预习PHP之路了……



## 展望

- 配合认证相关衍生更多功能
- 维护一个统一集中数据库，用于后面多平台认证
- 自动发布新闻内容（插件以肝完！）