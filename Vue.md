# Vue简介

前端三件套：

- HTML：决定网页结构
- CSS：决定显示效果
- JavaScript：决定网页功能（交互、数据显示等）

Vue 是一款用于构建用户界面的 JavaScript 框架——即只负责前端的数据展示以及页面渲染。它基于标准 HTML、CSS 和 JavaScript 构建，并提供了一套声明式的、组件化的编程模型，帮助高效地开发用户界面。

## MVVM模型

- 后端MVC：流程控制与视图渲染都由后端完成；
- 前端MVC：后端只负责接收请求并响应数据，前端负责渲染和显示逻辑；
- MVVM：后端只负责响应数据，前端接收到数据后设置到ViewModel，HTML自己到VM获取数据。

Vue正是基于MVVM模型，在HTML中使用时需要先创建VM，此后只需要将数据设置到VM中即可在HTML代码中使用“{{}}”来获取数据。

# Vue基本使用

*在HTML引入vue**（使用Vue2）**的JS库*

```html
<script src="https://cdn.jsdelivr.net/npm/vue@2/dist/vue.js"></script>
```

*创建Vue对象*

```html
<script type="text/javascript">
    var vm = new Vue({
        el: "#all",
        data: {
            str: "hello",
            stu: {
                age: 30,
                name: "rybin"
            }
        }
    })
</script>
```

*在HTML中引用VM中的数据*

```html
<body>
    <div id="all">
        从VM获取数据：{{str}}
        学生年龄：{{stu.age}}
    </div>
</body>
```

## 条件语句

```html
<label v-if="stu.age > 18">成年</label>
```

## 循环语句

```html
<div v-for="s, index in stus"></div>
```

## 绑定标签属性

```html
<input type="text" v-bind:value="str"/>
<!-- 可以简写成 -->
<input type="text" :value="str"/>
```

该语法能够将VM内的str数据放置在该位置。

## 双向绑定

```html
<input type="text" v-model:value="str"/>
<!-- 可以简写成 -->
<input type="text" v-model="str"/>
```

与前一个语法类似，但是当改变表单值时会同时改变VM内的数据，因此称作**双向绑定**。

# Vue实例

想要使用Vue的VM以及相关功能，必须创建Vue实例。

## Vue实例的生命周期

在Vue2中，Vue实例的生命周期大致有四个过程：

1. 创建实例（初始化data）
2. 将data数据渲染到页面
3. 当data数据发生改变时重新渲染
4. 销毁实例

![Vue 实例生命周期](assets/Vue.assets/lifecycle-17027859008932.png)

## 钩子函数

在Vue实例生命周期四个阶段的前后分别提供了钩子函数供开发者定制，Vue可以在到达对应时机时自动调用钩子函数。

- beforeCreate
- created
- beforeMount
- mounted
- beforeUpdate
- updated
- beforeDestroy
- destroyed

> 在上图中红色实线框就是钩子函数，指向的位置就是函数被执行的时机。

# 计算属性

data中的数据可以通过声明创建，也可以通过函数计算得到。

*创建Vue实例时使用computed属性即可创建一个计算属性*

```javascript
var vm = new Vue({
    el: "#all",
    data: {
        str: "hello"
    },
    computed: {
        strplus: function() {
            return this.str + ", world";
        }
    }
})
```

这里为strplus指定了一个函数，返回值是一个拼接的字符串。这意味着在HTML中引用strplus时都会获得被该函数计算得到的数据。**值得注意的是，函数中使用到的数据（例如这里的this.str）是引用的方式，即每当该数据改变，函数会重新计算返回值从而改变计算属性的值。**

# 侦听器

设置侦听器可以使得当某个VM中数据发生改变时，执行给定的侦听器函数。

*在创建Vue实例时使用watch属性即可为某一属性创建侦听器函数*

```javascript
watch: {
	str: function() {
        this.strplus = str;
    }
}
```

这个侦听器将会在每次str的值改变时，将strplus的值更改为相同的值。

# class绑定

```html
<!DOCTYPE html>
<html>
	<head>
		<meta charset="utf-8">
		<title></title>
		<script src="https://cdn.jsdelivr.net/npm/vue@2/dist/vue.js"></script>
		<style type="text/css">
			.mystyle1 {
				width: 200px;
				height: 100px;
				background: orange;
			}
			.mystyle3 {
				width: 200px;
				height: 100px;
				background: black;
			}
			.my-style2 {
				border-radius: 15px;
			}
		</style>
	</head>
	<body>
		<div id="all">
			<!-- 花括号内：样式名：布尔值，代表布尔值为真时包含该样式 -->
            <!-- 此处可以不使用单引号，但如果样式名有特殊字符则需要带单引号 -->
			<div v-bind:class="{mystyle1: style1, 'my-style2': style2}"></div>
			<!-- 数组语法：方括号内可以直接加载多个样式，但是注意单引号表示样式名，不带单引号则是data变量名 -->
			<div v-bind:class="['mystyle1', chooseStyle2]"></div>
			<!-- 三目运算符语法 -->
			<div v-bind:class="[choose ? chooseStyle3 : chooseStyle1]"></div>
		</div>
	</body>
	
	<script type="text/javascript">
		var vm = new Vue({
			el: "#all",
			data: {
				style1: true,
				style2: true,
				choose: false,
				chooseStyle1: "mystyle1",
				chooseStyle2: "my-style2",
				chooseStyle3: "mystyle3"
			}
		})
	</script>
</html>
```

