---
title: Vue基础学习笔记01
date: 2018-11-18 20:29:57
tags:
 - Vue
 - 前端
categories: Vue
---



Vue官方文档：https://cn.vuejs.org/v2/guide/ 

文档里什么都有！

<!-- more -->

# 准备开始

因为是基础学习，所以本例子都写在一个HTML中了。后面改变形式。

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<title>Vue基础</title>
</head>
<body>
	<div id="app">
		
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			
		})
	</script>
</body>
</html>
```

- 引入目录下的vue.min.js文件。
- 创建一个div id为app。
- 在script中创建一个变量为Vue对象，el是element的缩写，选择器写法选中刚才的div。接下来如要使用vue都要写在div内才有效。然后我们就开始了！！

# 模板语法

## 插值



### 文本

最直接数据绑定形式，使用双大括号形式。

```html
<div id="app">
		<p>{{msg}}</p>
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				msg:'简单插值'
			}
		})
	</script>
```

data中包含了我们声明使用的数据。

- p标签中的内容会显示为msg的值，而且是实时变化，也就是说msg变量改变 p标签内容相应变化
- `v-once`可以一次性插值，后面就撒手不管了（修改也不会变）

```html
<p v-once>{{msg}}</p>
```



### 原始html

如果直接用双大括号，在data中变量写原生HTML标签不会生效的。

使用`v-html="msg"`

```html
<div id="app">
		<p v-html="msg"></p>
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				msg:'<h1>简单插值</h1>'
			}
		})
	</script>
```

![rawhtml](https://upload-images.jianshu.io/upload_images/14657587-91e420b41618750d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 表达式

直接在中使用JavaScript简单表达式，比如

```html
{{ number + 1 }}

{{ ok ? 'YES' : 'NO' }}

{{ message.split('').reverse().join('') }}
```



## 指令

### 判断v-if

在标签中使用v-if，如下例子如果num为4就不显示出来了

```html
<div id="app">
		<p v-if="num > 4">{{msg}}</p>
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				msg:'iSk2y learn Vue',
				num: 5
			}
		})
	</script>
```



### 循环v-for

通常for用来列表渲染，**还有v-for比v-if优先级更高**

#### 遍历数组

```html
<div id="app">
		<ul>
			<li v-for="item in numList">{{item}}</li>
		</ul>		
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				numList:[1,2,3,4]
			}
		})
	</script>
```

#### 遍历对象

```html
<div id="app">
		<ul>
			<li v-for='item in obj'>{{item}}</li>
		</ul>
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				obj : {
					name:'Vue',
					version:'2.9'
				},
			}
		})
	</script>
```

#### 遍历对象数组

```html
<div id="app">
		<ul>
			<li v-for='item in objList'>{{item.name}}-{{item.version}}</li>
		</ul>
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				objList:[
					{name:'Python',version:3},
					{name:'PHP',version:7},
					{name:'Vue',version:2.9},
					{name:'Django',version:2.1}
				]
			}
		})
	</script>
```

![for遍历](https://upload-images.jianshu.io/upload_images/14657587-501adcfbe3ab49c0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




#### 多参数

##### index,value

可以提供两个参数 key键值和value

```html
<div id="app">
		<ul>
			<li v-for='(value,key) in obj'>{{key}} : {{value}}</li>
		</ul>
		
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				
				obj : {
					name:'Vue',
					version:'2.9'
				},
			}
		})
	</script>
```

结果为：

>- name : Vue
>- version : 2.9

##### index,key,value

还可以三个参数，index为索引，key为键值，value为值

```html
<div id="app">
		<ul>
			<li v-for='(value,key,index) in obj'>{{index}} : {{key}} : {{value}}</li>
		</ul>
		
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				
				obj : {
					name:'Vue',
					version:'2.9'
				},
			}
		})
	</script>
```

结果为：

>- 0 : name : Vue
>- 1 : version : 2.9

#### 数组更新检测

如果data中被遍历的数组发生了更新，使用正确的函数才会触发视图更新

- `push()`
- `pop()`
- `shift()`
- `unshift()`
- `splice()`
- `sort()`
- `reverse()`

![push](https://upload-images.jianshu.io/upload_images/14657587-b529d81597c78111.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




**注意：**由于 JavaScript 的限制，Vue 不能检测以下变动的数组：

1. 当你利用索引直接设置一个项时，例如：`vm.items[indexOfItem] = newValue`
2. 当你修改数组的长度时，例如：`vm.items.length = newLength`



#### 遍历范围

```html
<div id="app">
		<ul>
			<li v-for='i in 6'>{{ i }}</li>
		</ul>
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
		})
	</script>
```

结果为：

>- 1
>- 2
>- 3
>- 4
>- 5
>- 6

### 绑定v-bind

利用`v-bind`语法可以绑定元素的属性。如下`v-bind:href`缩写为`:href`

```html
<div id="app">
		<img v-bind:src="imgSrc">
		
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				imgSrc: 'https://cn.vuejs.org/images/logo.png'
			}
		})
	</script>
```

![vuebind.jpg](https://upload-images.jianshu.io/upload_images/14657587-69565b1da4ec79aa.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




### 事件v-on

> 这里要把函数写入methods内，这样才会调用成功。这个是固定的。

可以绑定JavaScript的原生事件，以click为例

```html
<div id="app">
		<p v-on:click='add'>{{num}}</p>
		
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				num: 66,
			},
			methods:{
				add(){
					this.num++;
					console.log('click p '+this.num);
				}
			}
		})
	</script>
```

![绑定](https://upload-images.jianshu.io/upload_images/14657587-49dbf9e8187d5f51.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




# 计算属性

模板内可以使用表达式，但是放入太多会让模板难以维护。把复杂逻辑放入计算属性内，只要数据一有更新变化，计算属性内的响应函数也会立即发生变化。



## 基础例子

要把计算属性写在compted中，然后**必须必须必须return**，默认是getter属性，可以自定setter属性

```html
<div id="app">
		<p>{{reverseMsg}}</p>
		
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				msg: 'I will be better',
			},
			computed:{
				reverseMsg(){
					return this.msg.split('').reverse().join('');
				}
			}
		})
	</script>
```


![计算属性](https://upload-images.jianshu.io/upload_images/14657587-374788bfb5236622.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



## 计算属性缓存VS方法

上面的例子用方法也能达到同样的效果

```html
<div id="app">
		<p>{{reversedMessage()}}</p>
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				msg: 'I will be better',
			},
			methods: {
				  reversedMessage: function () {
				    return this.msg.split('').reverse().join('')
				  }
				},
		})
	</script>
```

我们可以将同一函数定义为一个方法而不是一个计算属性。两种方式的最终结果确实是完全相同的。然而，不同的是**计算属性是基于它们的依赖进行缓存的**。只在相关依赖发生改变时它们才会重新求值。这就意味着只要 `message` 还没有发生改变，多次访问 `reversedMessage` 计算属性会立即返回之前的计算结果，而不必再次执行函数。

## setter

```html
<div id="app">
		<p>{{reverseMsg}}</p>
		
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				msg: 'I will be better',
			},
			computed:{
				reverseMsg:{
					get(){
						return this.msg.split('').reverse().join('');
					},
					set(){
						this.msg = '我就不设置你给的值'
					}
					
				}
			}
		})
	</script>
```

![compted2.jpg](https://upload-images.jianshu.io/upload_images/14657587-28dc1d065c626b30.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


发现我设置的是this.msg，然后模板中的就自动又调用reverseMsg计算属性逆转了字符串。


后续更新！！

