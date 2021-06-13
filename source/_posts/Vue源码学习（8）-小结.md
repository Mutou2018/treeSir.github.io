---
title: Vue源码学习（8）小结
date: 2021-06-13 11:10:13
tags: vue
cover: https://raw.githubusercontent.com/DaMu2018/cloudimg/main/data/vue.png
top_img: https://raw.githubusercontent.com/DaMu2018/cloudimg/main/data/ready-to-go.jpg
---
### 小结

*保持周更好困难，已经断更了，之前的存货快写完了，最近可能更新一下js基础知识，巩固基础😜*

Vue源码阅读大致告一段落，在阅读源码的时候，因为当前版本和书中介绍的版本已经有很多的不同，加上逻辑确实比较绕，所以有时候看到云里雾里的，但是总的来说，对于Vue有了更加深入的认识。

1.Vue创建时都是以render为主，包括目前常用的template写法，都是会经过编译转换成render类型的。

2.Vue组件，每一个都是Vue的实例。但是他们呈现树结构，根元素都是一个，且只能有一个。组件中使用`<template><template/>`语法里只能有一个`<div><div/>`知道了原因，因为解析的时候有一个根节点，根据作者解答是因为diff算法的原因。

3.知道了Vue创建时所走的流程，new Vue --> $mount --> compiler --> render --> vnode --> patch --> DOM

comilper -->parse -->生成ast-->optimize（标记静态节点） --> generate（codegen：将ast转换为可执行的代码render function）-->vnode

updateComponent --> _render -->createElememt --> _createElement --> createComponent --> 实例化vnode --> patch -->vnode --> DOM

createComponent --> initComponent --> componentInstance --> componnentInstanceForVNode -->  return new vnode.componentOptions.Ctor(options)  --> 合并options --> 执行$mount -->调用——_render方法

_update--> \_\_patch\_\_ -->web --> return patch --> createElm --> 创建真实DOM

4.了解了diff算法的比较方式，渐进对比比较。通过step在同一层级比较。

5.知道了组件注册使用vnode的占位的方式。

6.响应式的原理理顺了，了解了computed和watch都是基于watcher实现的，就是配置参数不一样，computed不同。

7.了解了slot和slot-scoped在源码上的区别，之前使用老是不能够区分。普通插槽就是在父组件实例上，作用域插槽是在子组件实例上的，但是他的参数是通过父组件的ast参数中的slotScoped属性赋值给子组件的。

8.keep-alive源码上了解了vue是如何实现缓存的，通过缓存vnode来保存dom，首次渲染相同，但是当命中缓存时，就会跳出原来的created，mounted等，返回之前存储的vnode，通过重新对slot解析，触发$forceUpdate，可以利用的生命周期就是actived。

9.emit父子传参的源码实现，就是emit执行函数是，回调函数在父组件，所以实现了父子组件的传参。

10.$forceUpdate 的逻辑非常简单，就是调用渲染 watcher 的 update 方法，update方法进入queueWatcher，nextTick执行flushSchedulerQueue。之后都会执行vm.\_update(vm.\_render)。

