---
title: Vue基础学习笔记02-实现markdown实时
date: 2018-11-19 14:01:59
tags:
 - Vue
 - 前端
categories: Vue
---

接着[上篇博客地址](http://isk2y.coding.me/2018/11/18/learn-Vue-01/) [简书地址](https://www.jianshu.com/p/983fbee793ab) 继续 

关于class和style样式绑定，v-model，v-if，v-show，v-on的笔记

<!-- more -->
# class与style绑定

操作元素的 class 列表和内联样式是数据绑定的一个常见需求。因为它们都是属性，所以我们可以用 `v-bind` 处理它们：只需要通过表达式计算出字符串结果即可。不过，字符串拼接麻烦且易错。因此，在将 `v-bind` 用于 `class` 和 `style` 时，Vue.js 做了专门的增强。表达式结果的类型除了字符串之外，还可以是对象或数组。

## 绑定class

### 对象表示

可以用对象来表示，active类的有无取决于isActive的[truthiness](https://developer.mozilla.org/zh-CN/docs/Glossary/Truthy) 。

```html
<div id="app" v-bind:class="{ active: isActive }">
		<p @click='changeActive'>{{msg}}</p>
		
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				msg: 'I will be better',
				isActive: true
			},
			methods:{
				changeActive(){
					this.isActive = !this.isActive;
                    
				}
			},
		})
	</script>
```

这个例子：点击p标签会触发changeActive事件，从而切换isActive的布尔值，达到active类添加和删除的效果。

#### 绑定计算属性

可以将class绑定在计算属性中，class中的对象也可以定义在data中的

```html
<style type="text/css">
		.active{
			background-color: rgb(127,199,211);
		}
		
	</style>
<div id="app" v-bind:class="ActiveClass">
		<p @click='changeActive'>{{msg}}</p>
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				msg: 'I will be better',
				isActive: true
			},
			computed:{
				ActiveClass(){
					return {active:!this.isActive}
				}
			}
		})
	</script>
```



### 数组表示

对象语法是用一个对象来表示，key为类名，value则为布尔值控制该类是否可以显示。而数组方式是把类名添加到数组中，从而添加该类。

```html
<style type="text/css">
		.active{
			background-color: rgb(127,199,211);
		}
		.myCenter{
			text-align: center;
		}
	</style>
<div id="app" v-bind:class="classList">
		<p @click='changeActive'>{{msg}}</p>
		
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				msg: 'I will be better',
				classList: []
			},
			methods:{
				changeActive(){
					this.classList.push("active");
					this.classList.push("myCenter")
				}
			},
			
		})
	</script>
```

该例，点击p标签触发绑定事件changeActive从而将active和myCenter push到classList变量中，v-bind:class因为classList变化从而渲染类的效果。

还可以把对象嵌入到数组中交叉运用。



## 内联样式绑定

可以直接绑定class，当然可以也可以对style样式做手脚了。

### 对象语法

`v-bind:style` 的对象语法十分直观——看着非常像 CSS，但其实是一个 JavaScript 对象。

```html
<div id="app" >
		<div v-bind:style="{ color: activeColor, fontSize: fontSize + 'px' }">{{msg}}</div>

	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				msg: 'I will be better',
				activeColor: 'rgb(127,199,211)',
  				fontSize: 30
			},	
		})
	</script>
```

可以直接绑定到一个对象上，如这样

```html
<div id="app" >
		<div v-bind:style="styleObj">{{msg}}</div>
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				msg: 'I will be better',
				styleObj:{
					color: 'rgb(127,199,211)',
  					fontSize: '30px'
				}
			},
		})
	</script>
```



### 数组语法

这个就是把上面的对象添加到数组中

```html
<div id="app" >
		<div v-bind:style="[styleObj, ]">{{msg}}</div>
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				msg: 'I will be better',
				styleObj:{
					color: 'rgb(127,199,211)',
  					fontSize: '30px'
				}
			},
		})
	</script>
```



# 条件渲染

上篇基础博文中初步讲过v-if的使用，这里再来记录一些

## v-if

简单if，还有else，当然了还有else-if，必须要紧挨着才能识别

```html
<div id="app" >
		<div v-if="status == 'good'">{{'if分支' + status}} </div>
		<div v-else-if="status == 'bad'">{{'else-if分支' + status}} </div>
		<div v-else="status == 'verygood'">{{'else分支' + status}} </div>
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				msg: 'I will be better',
				status: 'good'
			},
			
		})
	</script>
```

因为 `v-if` 是一个指令，所以必须将它添加到一个元素上。但是如果想切换多个元素呢？此时可以把一个 `<template>` 元素当做不可见的包裹元素，并在上面使用 `v-if`。最终的渲染结果将不包含 `<template>` 元素。

```html
<template v-if="ok">
  <h1>Title</h1>
  <p>Paragraph 1</p>
  <p>Paragraph 2</p>
</template>
```

## 用key管理可复用元素

Vue 会尽可能高效地渲染元素，通常会复用已有元素而不是从头开始渲染。这么做除了使 Vue 变得非常快之外，还有其它一些好处。例如，如果你允许用户在不同的登录方式之间切换：

```html
<div id="app" >
		<template v-if="loginType === 'username'">
		  <label>用户名</label>
		  <input placeholder="Enter your username">
		</template>
		<template v-else-if="loginType === 'email'">
		  <label>邮箱</label>
		  <input placeholder="Enter your email address">
		</template>
		<button @click='toggleType'>切换</button>
		
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				msg: 'I will be better',
				loginType: 'username',
				typeList:['username', 'email']
			},
			methods:{
				toggleType(){
					index = this.typeList.indexOf(this.loginType);
					if (index === this.typeList.length-1) {
						this.loginType = this.typeList[0];
						return;
					}else{
						this.loginType = this.typeList[index + 1];
					}
				}
			},
			
		})
	</script>
```

该例子中输入在input框中的内容将不会被清除，从而重用，直接显示label和input的placeholder值。

这是key不显示设置唯一才生效的，如果显式设置两个key不唯一，则在切换的时候会清除input框内容

```html
<template v-if="loginType === 'username'">
		  <label>用户名</label>
		  <input placeholder="Enter your username" key='username-input'>
		</template>
		<template v-else-if="loginType === 'email'">
		  <label>邮箱</label>
		  <input placeholder="Enter your email address" key='email-input'>
		</template>
```

模板这样修改就会清除input框了，但是label还是会重用的！

## v-show

`v-show`也可以根据条件来展示元素的，但是和`v-if`不同的是，它只是简单的切换元素的css属性display而已，在页面载入时，已经被渲染，但是根据条件可能没有显示，而if是完全惰性的，真正条件渲染。

这个是上面if的例子，用show改写，注意show没有if

```html
<div id="app" >
		<div v-show="status">{{msg}} </div>
		
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				msg: 'I will be better',
				status: false
			},		
		})
	</script>
```

初始设置为false，则载入页面时，我们将看不到div中的msg变量内容，但是整个dom元素中是存在的

![show](https://upload-images.jianshu.io/upload_images/14657587-563a6524c545b9f1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**注意：**`v-show` 不支持 `<template>` 元素，也不支持 `v-else`。



## v-if和v-show

`v-if` 是“真正”的条件渲染，因为它会确保在切换过程中条件块内的事件监听器和子组件适当地被销毁和重建。

`v-if` 也是**惰性的**：如果在初始渲染时条件为假，则什么也不做——直到条件第一次变为真时，才会开始渲染条件块。

相比之下，`v-show` 就简单得多——不管初始条件是什么，元素总是会被渲染，并且只是简单地基于 CSS 进行切换。

一般来说，`v-if` 有更高的切换开销，而 `v-show` 有更高的初始渲染开销。因此，如果需要非常频繁地切换，则使用 `v-show` 较好；如果在运行时条件很少改变，则使用 `v-if` 较好。



# 事件处理

## 监听事件

上节笔记中记录过v-on的相关用法，v-on还可以缩写为@

这个例子以click事件为例

```html
	<div id="app" >
		<div>{{msg + count}} </div>
		<button @click='addCount'>点击count加1</button>
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				msg: 'I will be better',
				count: 0,
			},
			methods:{
				addCount(){
					this.count += 1;
				}
			}
			
		})
	</script>
```

methods中是定义函数的地方，上例子中的addCount其实为`addCount:function(){}`，这里运用es6单例模式写法。

在`v-on:click='count + 1'`这样直接做操作也是可以的

### 传值

```html
	<div id="app" >
		<div>{{msg + count}} </div>
		<button @click='addCount(5)'>点击count加1</button>
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				msg: 'I will be better',
				count: 0,
			},
			methods:{
				addCount(num){
					this.count += num;
				}
			}
			
		})
	</script>
```



有时也需要在内联语句处理器中访问原始的 DOM 事件。可以用特殊变量 `$event` 把它传入方法：

```html
	<div id="app" >
		<div>{{msg + count}} </div>
		<button @click='addCount($event)'>点击count加1</button>
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				msg: 'I will be better',
				count: 0,
			},
			methods:{
				addCount(event){
					this.count += 1;
					console.log(event);
				}
			}
			
		})
	</script>
```



## 事件修饰符

在事件处理程序中调用 `event.preventDefault()` 或 `event.stopPropagation()` 是非常常见的需求。尽管我们可以在方法中轻松实现这点，但更好的方式是：方法只有纯粹的数据逻辑，而不是去处理 DOM 事件细节。

为了解决这个问题，Vue.js 为 `v-on` 提供了**事件修饰符**。之前提过，修饰符是由点开头的指令后缀来表示的。

- `.stop`
- `.prevent`
- `.capture`
- `.self`
- `.once`
- `.passive`

```html
<!-- 阻止单击事件继续传播 -->
<a v-on:click.stop="doThis"></a>

<!-- 提交事件不再重载页面 -->
<form v-on:submit.prevent="onSubmit"></form>

<!-- 修饰符可以串联 -->
<a v-on:click.stop.prevent="doThat"></a>

<!-- 只有修饰符 -->
<form v-on:submit.prevent></form>

<!-- 添加事件监听器时使用事件捕获模式 -->
<!-- 即元素自身触发的事件先在此处理，然后才交由内部元素进行处理 -->
<div v-on:click.capture="doThis">...</div>

<!-- 只当在 event.target 是当前元素自身时触发处理函数 -->
<!-- 即事件不是从内部元素触发的 -->
<div v-on:click.self="doThat">...</div>
```



## 按键修饰符

在监听键盘事件时，我们经常需要检查常见的键值。Vue 允许为 `v-on` 在监听键盘事件时添加按键修饰符：

```html
<!-- 只有在 `keyCode` 是 13 时调用 `vm.submit()` -->
<input v-on:keyup.13="submit">
```

记住所有的 `keyCode` 比较困难，所以 Vue 为最常用的按键提供了别名：

```html
<!-- 同上 -->
<input v-on:keyup.enter="submit">

<!-- 缩写语法 -->
<input @keyup.enter="submit">
```

全部的按键别名：

- `.enter`
- `.tab`
- `.delete` (捕获“删除”和“退格”键)
- `.esc`
- `.space`
- `.up`
- `.down`
- `.left`
- `.right`



# 表单输入绑定

使用`v-model`可以做到数据和元素双向绑定，在表单等输入控件内输入内容，会自动更新数据渲染元素。其本质不过是语法糖。

支持以下表单元素：

- `<input>`
- `<textarea>`
- `<select>`



## 文本框input

```html
	<div id="app" >
		<input v-model='message'>
		<p>{{message}}</p>
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				message: 'I will be better',
			},
		})
	</script>
```

### 封装前实现方式

`v-model`可以简单实现这个效果，但其实是语法糖（官网是说了），用绑定事件方法还是可以实现的，如下面

```html
	<div id="app" >
		<input @input='changeValue($event)'>
		<p>{{message}}</p>
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				message: 'I will be better',
			},
			methods:{
				changeValue(event){
					this.message = event.target.value;
					console.log(event.target.value);
				}
			}
		})
	</script>
```

![input实现方法](https://upload-images.jianshu.io/upload_images/14657587-367fc603d34cd963.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 多行文本textarea

```html
	<div id="app" >
		<textarea v-model='message'></textarea>
		<p>{{message}}</p>
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				message: 'I will be better',
			},
		})
	</script>
```

## 复选框

直接多个复选框吧，要绑定到一个数组

```html
	<div id="app" >
		<input type="checkbox"  value="Jack" v-model="message">
		  <label for="jack">Jack</label>
		  <input type="checkbox"  value="John" v-model="message">
		  <label for="john">John</label>
		  <input type="checkbox"  value="Mike" v-model="message">
		  <label for="mike">Mike</label>
		  <br>
		  <span>Checked names: {{ message }}</span>
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				message: [],
			},
		})
	</script>
```

![vuemodel.jpg](https://upload-images.jianshu.io/upload_images/14657587-b53b39f7f17fb21b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 单选框

上例改变

```html
	<div id="app" >
		<input type="radio"  value="Jack" v-model="message">
		  <label for="jack">Jack</label>
		  <input type="radio"  value="John" v-model="message">
		  <label for="john">John</label>
		  <input type="radio"  value="Mike" v-model="message">
		  <label for="mike">Mike</label>
		  <br>
		  <span>Checked names: {{ message }}</span>
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				message: '',
			},
		})
	</script>
```

## 选择框

还是上面的例子，继续改变

```html
	<div id="app" >
		<select v-model='message'>
			<option>Jack</option>
			<option>John</option>
			<option>Mike</option>
		</select>
		  <span>Checked names: {{ message }}</span>
	</div>
	<script src="vue.min.js"></script> 
	<script>
		var App = new Vue({
			el:"#app",
			data:{
				message: '',
			},
		})
	</script>
```



## 修饰符

### [`.lazy`](https://cn.vuejs.org/v2/guide/forms.html#lazy)

在默认情况下，`v-model` 在每次 `input` 事件触发后将输入框的值与数据进行同步 (除了[上述](https://cn.vuejs.org/v2/guide/forms.html#vmodel-ime-tip)输入法组合文字时)。你可以添加 `lazy` 修饰符，从而转变为使用 `change`事件进行同步：

```html
<!-- 在“change”时而非“input”时更新 -->
<input v-model.lazy="msg" >
```

### [`.number`](https://cn.vuejs.org/v2/guide/forms.html#number)

如果想自动将用户的输入值转为数值类型，可以给 `v-model` 添加 `number` 修饰符：

```html
<input v-model.number="age" type="number">
```

这通常很有用，因为即使在 `type="number"` 时，HTML 输入元素的值也总会返回字符串。如果这个值无法被 `parseFloat()` 解析，则会返回原始的值。

### [`.trim`](https://cn.vuejs.org/v2/guide/forms.html#trim)

如果要自动过滤用户输入的首尾空白字符，可以给 `v-model` 添加 `trim` 修饰符：

```html
<input v-model.trim="msg">
```



# markdown实时效果

利用v-model可以简单实现markdown书写实现效果，当然我们要引入markdown的语法解析js。其实markdown就是提供了我们简单的语法，然后转换成HTML代码加上部分的样式。

我这里不用import的方法引入了，直接引入js文件

> marked：https://www.npmjs.com/package/marked
>
> npm中解析markdown的包

渣渣代码如下：

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<title>Vue基础</title>
	<style type="text/css">
		#app{
			top:0;
			bottom: 0;
			left: 0;
			right: 0;
			position: absolute;
		}
		.markleft{
			height: 100%;
			width: 50%;
			float: left;
		}
		textarea{
			height: 100%;
			width: 100%;
			background: black;
			color: white;
			overflow: scroll;
			font-family: 'Monaco', 'Menlo';
    		
		}
		.showright{
			height: 100%;
			width: 50%;
			float: right;
			overflow: scroll;
			
		}
	</style>
</head>
<body>
	<div id="app" >
		<div class="markleft">
			<textarea v-model='message'></textarea>
		</div>
		<div class="showright" v-html='marked(message)'></div>
	</div>
	<script src="vue.min.js"></script> 
	<script src="https://cdn.jsdelivr.net/npm/marked/marked.min.js"></script>
	<script>

		var App = new Vue({
			el:"#app",
			data:{
				message: 'I will be better',
			},
			methods:{
				changeValue(event){
					this.message = event.target.value;
					console.log(event.target.value);
				}
			}
		})
	</script>
</body>
</html>
```

![markdown](https://upload-images.jianshu.io/upload_images/14657587-d7b720ccd21effdb.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
