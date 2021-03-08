---
title: Vue源码学习（1）
date: 2020-03-16 05:49:17
tags: vue
cover: https://raw.githubusercontent.com/DaMu2018/cloudimg/main/data/vue.png
top_img: https://raw.githubusercontent.com/DaMu2018/cloudimg/main/data/ready-to-go.jpg
description: vue源码学习，了解vue是什么，源文件入口
---
# vue源码阅读笔记（1）

> 基于vue技术揭秘结合自己的相关认识

1.src源码目录

```
src 
├── compiler # 编译
├── core  # 核⼼ 
├── platforms # 不同平台（web，weex） 
├── server # 服务端渲染
├── sfc # 解析.vue 文件
├── shared # 常用的公共代码方法
```

2.源码分析

> platforms/entry-runtime-with-compiler.js

plateforms中有不同类型的方式，在vue-cli中有个询问compiler+runtime的，就是大多数选择的方式，compiler比单独runtime多了template的边编译，没有的话则不能使用tempalte的语法。

```
new Vue({
el:'#app',
})
```

el就是document.querySelector的语法糖，在其中有个warn的警告，当没有挂载dom或者dom没有查找到时的提示，在core/util/debug.js中。

在该js中还有对template的验证和报错提示。

这个文件就导出Vue，当使用import Vue from ‘vue’时，就是这个入口来初始化Vue。

Vue的核心文件根据import引入路径， core/instance/index.js。

（引入路径：/platforms/runtime/index.js --> /core/index -->  /core/instance/index.js）

> core/instance/index.js

这个是vue的核心文件，包括mixin，state，lifecycle等文件。这里没有使用class方法，而是使用的function 定义的Vue，且很多xxxMixin的函数，这些都是给Vue的prototype扩展方法。

Vue在global-api中定义了扩展方法

```
 Vue.util = {

  warn,

  extend,

  mergeOptions,

  defineReactive

 }
 ...
```

这里Vue本质上是一个用function实现的的class。

3.数据驱动

MVVM：model-view-modelview相对于早期使用JQuery直接操作DOM来说，Vue等现代框架更注重数据，而不是操作DOM，修改数据就会直接反应在view上。Vue的等框架没有直接操作DOM来的更快，对于简单的需求来说，甚至是拖累，但是Vue、react的等框架提供了更加相对合理的操作DOM方式，在对于DOM开销大的时候，就体现出了这些框架的优越性。

> DOM变成了数据的映射

这就是VDOM。

3.1 new Vue

在init.js中expose real self

```
 vm._self = vm

  initLifecycle(vm)

  initEvents(vm)

  initRender(vm)

  callHook(vm, 'beforeCreate')

  initInjections(vm) // resolve injections before data/props

  initState(vm)

  initProvide(vm) // resolve provide after data/props

  callHook(vm, 'created') 
```

这里干了初始化生命周期，初始化事件中心，初始化渲染等

```
vm.$mount(vm.$options.el)
```

最后挂载el。

这里有个callhook，里面有beforeCreate，created，这些操作都在vm.$mount之前。需要结合生命周期更加清楚。

3.2 $mount

$mount的定义在entry-runtime-compiler.js中在Vue的原型链上。使用template最后还是会被compileToFunctions编译成render函数。

$mount(el,hydrating),hydrating这个参数是表示在服务器端相关的参数。在entry-runtime中是只有render，没有template的，在这个文件中引用的Vue来自runtime/index.js，其实在entry-runtime-with-compiler中

```
const mount = Vue.prototype.$mount
```

也是将原型链上$mount存在了mount中。公共的挂载方法在mountComponent()函数调用。这里面有很多个生命周期的定义，beforMount,mounted,beforeUpdate,activated,deactivated,beforeDestroy,destroyed这些生命周期的调用。在mountComponent中有beforMount和mounted这两个生命周期，其中：

```
const vnode = vm._render()
```

定义了vnode，在Watcher实例中定义了beforeUpdate生命周期，监测数据更新以及初始化执行回调函数。

```
if (vm.$vnode == null) { //通过 == null来表示当前是Vue的根实例
    vm._isMounted = true
    callHook(vm, 'mounted')
}
```

在mountComponent中主要方法是vm._render,vm. _update。



3.3 render

在render.js中定义了_render函数。

> 类型：(createElement: () => VNode) => VNode
> 该渲染函数接收一个 createElement 方法作为第一个参数用来创建 VNode。

官网上说createElement方法作为第一个参数来创建VNode。

```
vnode = render.call(vm._renderProxy, vm.$createElement)
```

createElement对应vm.$createElement

```
vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)
```

createElement(vm, a, b, c, d, false)中5个参数对应，args order: tag, data, children, normalizationType, alwaysNormalize。

render函数调用createElement()方法并返回VNode。

之后继续学习VNode