---
title: 简单解读Xiuno BBS之流程
date: 2018-12-11 22:26:32
tags:
 - PHP
 - xiuno BBS
categories: PHP
---

很久没有碰PHP代码额，作为阅读记录吧。

<!-- more -->

# XiunoBBS 

- Xiuno BBS 4.0 开发手册：https://www.kancloud.cn/xiuno/bbs4
- XiunoPHP 开发手册：<http://www.kancloud.cn/xiuno/xiunophp> 
- Xiuno BBS 4.0 二次开发文档：https://bbs.xiuno.com/thread-13108.htm



# 由来

最近因为特殊原因想建立一个社区（论坛）有某些作用，机缘巧合下找到xiuno bbs，看过上面的开发手册后，感觉很有利于二次开发，所以好久没看PHP的我重新捡起来**预习**下，记录下整个流程加载。

手册中很多内容写的很详细，不做重新复述。

# 结构

## XiunoBBS目录结构

v4.0

```
admin/                             -- 后台管理目录
    route                          -- 后台路由
    view                           -- 后台模板
    index.php                  -- 后台入口
    index.inc.php             -- 入口包含的代码段
    admin.func.php         -- 后台依赖的函数
    menu.conf.php         -- 后台配置文件
conf/                              -- 全站配置文件
    conf.php                   -- 配置文件
    conf.default.php       -- 默认配置文件
    attach.conf.php        -- 附件配置文件
    smtp.conf.php          -- 发送邮件的配置文件
install/                            -- 安装目录
lang/                              -- 语言包
log/                                -- 日志目录，按照天存放日志
model/                           -- 数据处理的函数文件目录
plugin/                           -- 插件目录，一个插件一个目录
route/                             -- 路由目录，业务逻辑处理
tmp/                               -- 临时文件存放目录，插件和代码合并后的文件存放于此
upload/                          -- 上传目录
view/                              -- 前端模板目录
robots.txt                    -- 屏蔽蜘蛛的配置文件
.htaccess                      -- Apache URL-Rewrite 文件
index.inc.php               --  前端入口包含代码段
model.inc.php             -- Model 包含目录
index.php                    -- 前台程序入口
```



## hook位置

```
// hook xiunophp_include_before.php		xiunophp包含前
// hook xiunophp_include_after.php 		xiunophp包含完成后
// hook index_inc_start.php			   index配置开始
// hook index_inc_end.php			   index配置结束
// hook index_inc_route_before.php		路由开始前
……
```

hook的位置太多了。。



## override位置

```
index.inc.php
view/htm/*.htm
route/*.php
model/*.php
admin/view/htm/*.htm
admin/route/*.php
admin/index.inc.php
admin/menu.conf.php
lang/*.php
```

这些都可以被override



# 加载

在 Xiuno BBS 4.0 当中，采用的单入口设计，全部从 index.php 进。
所有的 xxx-xxx.htm 都通过 Web Server 转发到了 index.php?route-action.htm。
由 route 目录下对应的 php 文件进行处理（Controller 层）。
model 则为数据处理目录（Model 层）。
view 为 js css font 等负责显示的文件 目录（View 层）。

![img](https://box.kancloud.cn/97b15b3e8ec4f01d181082317e8888f3_426x320.png)



## 入口

文档中所说是从index.php唯一入口进入，

```php
<?php

// 0: 线上模式; 1: 调试模式; 2: 插件开发模式;
!defined('DEBUG') AND define('DEBUG', 0);
define('APP_PATH', dirname(__FILE__).'/'); // __DIR__
!defined('ADMIN_PATH') AND define('ADMIN_PATH', APP_PATH.'admin/');
!defined('XIUNOPHP_PATH') AND define('XIUNOPHP_PATH', APP_PATH.'xiunophp/');

$conf = (@include APP_PATH.'conf/conf.php') OR exit('<script>window.location="install/"</script>');

// 兼容 4.0.3 的配置文件	
!isset($conf['user_create_on']) AND $conf['user_create_on'] = 1;
!isset($conf['logo_mobile_url']) AND $conf['logo_mobile_url'] = 'view/img/logo.png';
!isset($conf['logo_pc_url']) AND $conf['logo_pc_url'] = 'view/img/logo.png';
!isset($conf['logo_water_url']) AND $conf['logo_water_url'] = 'view/img/water-small.png';
$conf['version'] = '4.0.4';		// 定义版本号！避免手工修改 conf/conf.php

// 转换为绝对路径，防止被包含时出错。
substr($conf['log_path'], 0, 2) == './' AND $conf['log_path'] = APP_PATH.$conf['log_path']; 
substr($conf['tmp_path'], 0, 2) == './' AND $conf['tmp_path'] = APP_PATH.$conf['tmp_path']; 
substr($conf['upload_path'], 0, 2) == './' AND $conf['upload_path'] = APP_PATH.$conf['upload_path']; 

$_SERVER['conf'] = $conf;

if(DEBUG > 1) {//根据debug模式 判断加载什么xiunophp。
	include XIUNOPHP_PATH.'xiunophp.php';
} else {
	include XIUNOPHP_PATH.'xiunophp.min.php';
}


// 加载插件
include APP_PATH.'model/plugin.func.php';
include _include(APP_PATH.'model.inc.php');
include _include(APP_PATH.'index.inc.php');

//file_put_contents((ini_get('xhprof.output_dir') ? : '/tmp') . '/' . uniqid() . '.xhprof.xhprof', serialize(xhprof_disable()));

?>
```

1. 定义了APP_PATH，ADMIN_PATH，XIUNOPHP_PATH的目录
2. 包含APP_PATH应用目录下的配置目录下的配置文件conf/conf.php，这里就代表引入了配置
3. 将log、upload、tmp目录转换成绝对路径
4. 把配置conf赋值给server数组
5. 然后加载xiunophp，待会再看里面有哪些东西
6. 包含plugin.func.php，加载插件某些函数
7. 包含model.inc.php 
8. 包含index.inc.php



最后这几个包含都挺有内容的，挨个来查看



### xiunophp

引入了缓存类、数据库支持类、还有一些自定义封装的函数，然后初始化某些内容，具体可以看代码中如何书写。

最后往server超全局变量中存储了这些内容

```php
// 保存到超级全局变量，防止冲突被覆盖。
$_SERVER['starttime'] = $starttime;
$_SERVER['time'] = $time;
$_SERVER['ip'] = $ip;
$_SERVER['longip'] = $longip;
$_SERVER['useragent'] = $useragent;
$_SERVER['conf'] = $conf;
$_SERVER['lang'] = $lang;
$_SERVER['errno'] = $errno;
$_SERVER['errstr'] = $errstr;
$_SERVER['method'] = $method;
$_SERVER['ajax'] = $ajax;
$_SERVER['get_magic_quotes_gpc'] = $get_magic_quotes_gpc;
$_SERVER['db'] = $db;
$_SERVER['cache'] = $cache;
```




### plugin.func.php

plugin.func.php这个文件中定义了有关插件的各种函数，在这里不一一列举。是为后面加载插件做准备。

其中有部分函数经常会调用

```php
function _include($srcfile) {
	global $conf;
	// 合并插件，存入 tmp_path
	$len = strlen(APP_PATH);
	$tmpfile = $conf['tmp_path'].substr(str_replace('/', '_', $srcfile), $len);
	if(!is_file($tmpfile) || DEBUG > 1) {
		// 开始编译
		$s = plugin_compile_srcfile($srcfile);
		
		// 支持 <template> <slot>
		$g_include_slot_kv = array();
		for($i = 0; $i < 10; $i++) {
			$s = preg_replace_callback('#<template\sinclude="(.*?)">(.*?)</template>#is', '_include_callback_1', $s);
			if(strpos($s, '<template') === FALSE) break;
		}
		file_put_contents_try($tmpfile, $s);
		
		$s = plugin_compile_srcfile($tmpfile);
		file_put_contents_try($tmpfile, $s);
		
	}
	return $tmpfile;
}


// 编译源文件，把插件合并到该文件，不需要递归，执行的过程中 include _include() 自动会递归。
function plugin_compile_srcfile($srcfile) {
	global $conf;
	// 判断是否开启插件
	if(!empty($conf['disabled_plugin'])) {
		$s = file_get_contents($srcfile);
		return $s;
	}
	
	// 如果有 overwrite，则用 overwrite 替换掉
	$srcfile = plugin_find_overwrite($srcfile);
	$s = file_get_contents($srcfile);
	
	// 最多支持 10 层
	for($i = 0; $i < 10; $i++) {
		if(strpos($s, '<!--{hook') !== FALSE || strpos($s, '// hook') !== FALSE) {
			$s = preg_replace('#<!--{hook\s+(.*?)}-->#', '// hook \\1', $s);
			$s = preg_replace_callback('#//\s*hook\s+(\S+)#is', 'plugin_compile_srcfile_callback', $s);
		} else {
			break;
		}
	}
	return $s;
}

// 只返回一个权重最高的文件名
function plugin_find_overwrite($srcfile) {
	//$plugin_paths = glob(APP_PATH.'plugin/*', GLOB_ONLYDIR);
	
	$plugin_paths = plugin_paths_enabled();
	
	$len = strlen(APP_PATH);
	/*
	// 如果发现插件目录，则尝试去掉插件目录前缀，避免新建的 overwrite 目录过深。
	if(strpos($srcfile, '/plugin/') !== FALSE) {
		preg_match('#'.preg_quote(APP_PATH).'plugin/\w+/#i', $srcfile, $m);
		if(!empty($m[0])) {
			$len = strlen($m[0]);
		}
	}*/
	
	$returnfile = $srcfile;
	$maxrank = 0;
	foreach($plugin_paths as $path=>$pconf) {
		
		// 文件路径后半部分
		$dir = file_name($path);
		$filepath_half = substr($srcfile, $len);
		$overwrite_file = APP_PATH."plugin/$dir/overwrite/$filepath_half";
		if(is_file($overwrite_file)) {
			$rank = isset($pconf['overwrites_rank'][$filepath_half]) ? $pconf['overwrites_rank'][$filepath_half] : 0;
			if($rank >= $maxrank) {
				$returnfile = $overwrite_file;
				$maxrank = $rank;
			}
		}
	}
	return $returnfile;
}

// 可用的插件 必须enable和installed 这个在conf.json中有配置
function plugin_paths_enabled() {
	static $return_paths;
	if(empty($return_paths)) {
		$return_paths = array();
		$plugin_paths = glob(APP_PATH.'plugin/*', GLOB_ONLYDIR);
		if(empty($plugin_paths)) return array();
		foreach($plugin_paths as $path) {
			$conffile = $path."/conf.json";
			if(!is_file($conffile)) continue;
			$pconf = xn_json_decode(file_get_contents($conffile));
			if(empty($pconf)) continue;
			if(empty($pconf['enable']) || empty($pconf['installed'])) continue;
			$return_paths[$path] = $pconf;
		}
	}
	return $return_paths;
}

function plugin_compile_srcfile_callback($m) {
	static $hooks;
	if(empty($hooks)) {
		$hooks = array();
		$plugin_paths = plugin_paths_enabled();
		
		//$plugin_paths = glob(APP_PATH.'plugin/*', GLOB_ONLYDIR);
		foreach($plugin_paths as $path=>$pconf) {
			$dir = file_name($path);
			$hookpaths = glob(APP_PATH."plugin/$dir/hook/*.*"); // path
			if(is_array($hookpaths)) {
				foreach($hookpaths as $hookpath) {
					$hookname = file_name($hookpath);
					$rank = isset($pconf['hooks_rank']["$hookname"]) ? $pconf['hooks_rank']["$hookname"] : 0;
					$hooks[$hookname][] = array('hookpath'=>$hookpath, 'rank'=>$rank);
				}
			}
		}
		foreach ($hooks as $hookname=>$arrlist) {
			$arrlist = arrlist_multisort($arrlist, 'rank', FALSE);
			$hooks[$hookname] = arrlist_values($arrlist, 'hookpath');
		}
		
	}
	
	$s = '';
	$hookname = $m[1];
	if(!empty($hooks[$hookname])) {
		$fileext = file_ext($hookname);
		foreach($hooks[$hookname] as $path) {
			$t = file_get_contents($path);
			if($fileext == 'php' && preg_match('#^\s*<\?php\s+exit;#is', $t)) {
				// 正则表达式去除兼容性比较好。
				$t = preg_replace('#^\s*<\?php\s*exit;(.*?)(?:\?>)?\s*$#is', '\\1', $t);
				
				/* 去掉首尾标签
				if(substr($t, 0, 5) == '<?php' && substr($t, -2, 2) == '?>') {
					$t = substr($t, 5, -2);		
				}
				// 去掉 exit;
				$t = preg_replace('#\s*exit;\s*#', "\r\n", $t);
				*/
			}
			$s .= $t;
		}
	}
	return $s;
}
```

- \_include：将srcfile文件编译等操作后存入tmp_path中，
- plugin_compile_srcfile：编译源文件，寻找overwrite和hook的位置并合成最终文件
- plugin_find_overwrite：寻找overwrite，只返回一个权重最高的文件名
- plugin_compile_srcfile_callback：hook是在这个回调函数中替换的
  - glob — 寻找与模式匹配的文件路径
  - array_multisort — 对多个数组或多维数组进行排序
  - array_values — 返回数组中所有的值
- plugin_paths_enabled：可用的插件，是根据插件下的conf.json中判别的



![返回的plugin_paths](https://upload-images.jianshu.io/upload_images/14657587-0ba8c2ebaf65839d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![返回的hooks数组](https://upload-images.jianshu.io/upload_images/14657587-733613fbc66cfccd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### model.inc.php

model.inc.php这个文件中定义了待会要载入的model等许多

```php
if(DEBUG) {
	foreach ($include_model_files as $model_files) {
		include _include($model_files);
	}
} else {
	
	$model_min_file = $conf['tmp_path'].'model.min.php';
	$isfile = is_file($model_min_file);
	if(!$isfile) {
		$s = '';
		foreach($include_model_files as $model_files) {
			
			// 压缩后不利于调试，有时候碰到未结束的 php 标签，会暴 500 错误
			//$s .= php_strip_whitespace(_include($model_files));

			$t = file_get_contents(_include($model_files));
			$t = trim($t);
			$t = ltrim($t, '<?php');
			$t = rtrim($t, '?>');
			$s .= "<?php\r\n".$t."\r\n?>";

		}
		$r = file_put_contents($model_min_file, $s);
		unset($s);
	}
	include $model_min_file;
}

```



根据debug来判断是否要用压缩版的model.min.php，一般线上模式都是用min版，响应更快

这边主要是遍历$include_model_files这个数组，将各个model去_include编译放置到临时目录下，然后trim下。然后把所有的都编译到model.min.php中

这里注意下，我开始一直有个疑问，那如果加入安装插件，按照这个代码逻辑如果临时目录下存在该已经编译过的文件了，那怎么插件会安装完就能用呢？

这个流程我反复看了好久，后来想到，肯定是安装的时候动手脚了把，肯定显式执行删除了。。果然翻看后看到了删除编译文件的操作

```php
// 清空插件的临时目录
function plugin_clear_tmp_dir() {
	global $conf;
	rmdir_recusive($conf['tmp_path'], TRUE);
	xn_unlink($conf['tmp_path'].'model.min.php');
}
```

至于如果是自己开发插件，把debug改成2就可以了，这样每次都会自动去查找发现overwrite和hook。





### index.inc.php

这个是index.php中最后include的一个了，之前那些都是初始化和引入文件。都是为了后面做准备，这个文件中应该会具体涉及到具体逻辑和路由转发功能了，否则怎么访问到页面对吧？

```php
<?php

!defined('DEBUG') AND exit('Access Denied.');

// hook index_inc_start.php

$sid = sess_start();

// 语言 / Language
$_SERVER['lang'] = $lang = include _include(APP_PATH."lang/$conf[lang]/bbs.php");

// 用户组 / Group
$grouplist = group_list_cache();

// 支持 Token 接口（token 与 session 双重登陆机制，方便 REST 接口设计，也方便 $_SESSION 使用）
// Support Token interface (token and session dual match, to facilitate the design of the REST interface, but also to facilitate the use of $_SESSION)
$uid = intval(_SESSION('uid'));
empty($uid) AND $uid = user_token_get() AND $_SESSION['uid'] = $uid;
$user = user_read($uid);

$gid = empty($user) ? 0 : intval($user['gid']); // 或者当前用户所在组id
$group = isset($grouplist[$gid]) ? $grouplist[$gid] : $grouplist[0]; // 获取当前用户所在组

// 版块 / Forum
$fid = 0;
$forumlist = forum_list_cache();
$forumlist_show = forum_list_access_filter($forumlist, $gid);	// 有权限查看的板块 / filter no permission forum
$forumarr = arrlist_key_values($forumlist_show, 'fid', 'name');

// 头部 header.inc.htm 
$header = array(
	'title'=>$conf['sitename'],
	'mobile_title'=>'',
	'mobile_link'=>'./',
	'keywords'=>'', // 搜索引擎自行分析 keywords, 自己指定没用 / Search engine automatic analysis of key words, so keep it empty.
	'description'=>strip_tags($conf['sitebrief']),
	'navs'=>array(),
);

// 运行时数据，存放于 cache_set() / runtime data
$runtime = runtime_init();

// 检测站点运行级别 / restricted access
check_runlevel();

// 全站的设置数据，站点名称，描述，关键词
// $setting = kv_get('setting');

$route = param(0, 'index');

// hook index_inc_route_before.php

if(!defined('SKIP_ROUTE')) {
	
	// 按照使用的频次排序，增加命中率，提高效率
	// According to the frequency of the use of sorting, increase the hit rate, improve efficiency
	switch ($route) {
		// hook index_route_case_start.php
		case 'index': 	include _include(APP_PATH.'route/index.php'); 	break;
		case 'thread':	include _include(APP_PATH.'route/thread.php'); 	break;
		case 'forum': 	include _include(APP_PATH.'route/forum.php'); 	break;
		case 'user': 	include _include(APP_PATH.'route/user.php'); 	break;
		case 'my': 	include _include(APP_PATH.'route/my.php'); 	break;
		case 'attach': 	include _include(APP_PATH.'route/attach.php'); 	break;
		case 'post': 	include _include(APP_PATH.'route/post.php'); 	break;
		case 'mod': 	include _include(APP_PATH.'route/mod.php'); 	break;
		case 'browser': include _include(APP_PATH.'route/browser.php'); break;
		// hook index_route_case_end.php
		default: 
			// hook index_route_case_default.php
			include _include(APP_PATH.'route/index.php'); 	break;
			//http_404();
			/*
			!is_word($route) AND http_404();
			$routefile = _include(APP_PATH."route/$route.php");
			!is_file($routefile) AND http_404();
			include $routefile;
			*/
	}
}

// hook index_inc_end.php

?>
```

很多操作代码中的注释写的很清楚了，大赞作者！！



# 路由、Controller 层

index.inc.php最后的就是路由控制了，默认是route/index.php了，$route是由param这个xiunophp中的封装函数解析后得来的。

然后要注意的是这里也是执行\_include的，

```php
include _include(APP_PATH.'route/index.php'); 	break;
```



会先判断是否在tmp目录下存在这个文件，如果不存在就重新编译一份。

在编译的时候会找到所有的overwrite和hook并按照rank排列好，overwrite是按照权重找最高的，hook按照rank，然后最终合成为一个文件放在tmp这个临时文件中去了，

然后我们回到正题，这个route的index.php干了什么

```php
<?php

/*
* Copyright (C) 2015 xiuno.com
*/

!defined('DEBUG') AND exit('Access Denied.');

// hook index_start.php

$page = param(1, 1);
$order = $conf['order_default'];
$order != 'tid' AND $order = 'lastpid';
$pagesize = $conf['pagesize'];
$active = 'default';

// 从默认的地方读取主题列表
$thread_list_from_default = 1;

// hook index_thread_list_before.php
if($thread_list_from_default) {
	$fids = arrlist_values($forumlist_show, 'fid');
	$threads = arrlist_sum($forumlist_show, 'threads');
	$pagination = pagination(url("$route-{page}"), $threads, $page, $pagesize);
	
	// hook thread_find_by_fids_before.php
	$threadlist = thread_find_by_fids($fids, $page, $pagesize, $order, $threads);
}

// 查找置顶帖
if($order == $conf['order_default'] && $page == 1) {
	$toplist3 = thread_top_find(0);
	$threadlist = $toplist3 + $threadlist;
}

// 过滤没有权限访问的主题 / filter no permission thread
thread_list_access_filter($threadlist, $gid);

// SEO
$header['title'] = $conf['sitename']; 				// site title
$header['keywords'] = ''; 					// site keyword
$header['description'] = $conf['sitebrief']; 			// site description
$_SESSION['fid'] = 0;

// hook index_end.php

include _include(APP_PATH.'view/htm/index.htm');

?>
```



注释仍然写得很清楚，再大赞作者

最后包含了_include(APP_PATH.'view/htm/index.htm')这个，这里就到了视图最后显示给用户的阶段了





# 视图view

接着这个示例来看index.htm

```html
<?php include _include(APP_PATH.'view/htm/header.inc.htm');?>

<!--{hook index_start.htm}-->

<div class="row">
	<div class="col-lg-9 main">
		<!--{hook index_main_start.htm}-->
		<div class="card card-threadlist">
			<div class="card-header">
				<ul class="nav nav-tabs card-header-tabs">
					<li class="nav-item">
						<a class="nav-link <?php echo $active == 'default' ? 'active' : '';?>" href="./<?php echo url("$route");?>"><?php echo lang('new_thread');?></a>
					</li>
					<!--{hook index_thread_list_nav_item_after.htm}-->
				</ul>
			</div>
			<div class="card-body">
				<ul class="list-unstyled threadlist mb-0">
					<!--{hook index_threadlist_before.htm}-->
					<?php include _include(APP_PATH.'view/htm/thread_list.inc.htm');?>
					<!--{hook index_threadlist_after.htm}-->
				</ul>
			</div>
		</div>
		
		<?php include _include(APP_PATH.'view/htm/thread_list_mod.inc.htm');?>
		
		<!--{hook index_page_before.htm}-->
		<nav class="my-3"><ul class="pagination justify-content-center flex-wrap"><?php echo $pagination; ?></ul></nav>
		<!--{hook index_page_end.htm}-->
	</div>
	<div class="col-lg-3 d-none d-lg-block aside">
		<a role="button" class="btn btn-primary btn-block mb-3" href="<?php echo url('thread-create-'.$fid);?>"><?php echo lang('thread_create_new');?></a>
		<!--{hook index_site_brief_before.htm}-->
		<div class="card card-site-info">
			<!--{hook index_site_brief_start.htm}-->
			<div class="m-3">
				<h5 class="text-center"><?php echo $conf['sitename'];?></h5>
				<div class="small line-height-3"><?php echo $conf['sitebrief'];?></div>
			</div>
			<div class="card-footer p-2">
				<table class="w-100 small">
					<tr align="center">
						<td>
							<span class="text-muted"><?php echo lang('threads');?></span><br>
							<b><?php echo $runtime['threads'];?></b>
						</td>
						<td>
							<span class="text-muted"><?php echo lang('posts');?></span><br>
							<b><?php echo $runtime['posts'];?></b>
						</td>
						<td>
							<span class="text-muted"><?php echo lang('users');?></span><br>
							<b><?php echo $runtime['users'];?></b>
						</td>
						<?php if($runtime['onlines'] > 0) { ?>
						<td>
							<span class="text-muted"><?php echo lang('online');?></span><br>
							<b><?php echo $runtime['onlines'];?></b>
						</td>
						<?php } ?>
					</tr>
				</table>
			</div>
			<!--{hook index_site_brief_end.htm}-->
		</div>
		<!--{hook index_site_brief_after.htm}-->
	</div>
</div>

<!--{hook index_end.htm}-->

<?php include _include(APP_PATH.'view/htm/footer.inc.htm');?>

<script>
$('li[data-active="fid-0"]').addClass('active');
</script>

<!--{hook index_js.htm}-->
```

在上个部分路由控制层最后包含这个文件又执行了_include这个函数，所以这其中的hook一样到最后都是会被插件中对应的hook内容增加上去。

而且这个文件中还包含了header.inc.htm和footer.inc.htm内容，所以\_include会嵌套包含最终组合编译成一个文件放在tmp的临时目录中，等到下次访问就直接会访问那个目录下的文件，速度就会提高很多了！！

然后将内容返回给前端浏览器渲染给我们了。



# 收获

暂时先简单记录下这个流程，没有给出太多的细节，因为看代码更直观。