---
title: Vue源码学习（2）
date: 2020-03-18 07:41:53
tags: vue
cover: https://raw.githubusercontent.com/DaMu2018/cloudimg/main/data/vue.png
top_img: https://raw.githubusercontent.com/DaMu2018/cloudimg/main/data/ready-to-go.jpg
---
# vue源码阅读笔记（2）

1、VNode

/core/vdom/vnode.js由vnode的定义。这里面定义了tag，data，child等参数。VNode映射到真实的DOM实际上要经历create，diff，patch等过程。

而VNode的create创建方法则是定义在core/vdom/create-element中的

1.1 VNode create

> createElement方法实际上是对_createElement方法的封装

createElement的参数由context（VNode上下文环境），tag（标签），data（VNodeData类型的数据），children（VNode的子节点），normalizationType（子节点规范的类型）。

因为子节点获取的类型为any，意味着可以获取任意类型， 所以在子节点需要normalizationType规范子节点，这里在core/vdom/helpers/normalize-children有两个方法**simpleNormalizeChildren**，**normalizeChildren**，simpleNormalizeChildren这个方法只是将子节点的数据扁平化，子节点数组只有1层深度

normalizeChildren这个方法则是处理手写的render函数，jsx语法。这里面最终都将把children交给createTextVNode，返回VNode类型。

> 经过对 children 的规范化， children 变成了⼀个类型为 VNode 的 Array。

createElement创建VNode的过程，每个VNode有children，children每个元素也是一个VNode，就形成了VNode Tree，这样就很好的描述了DOM Tree。

vm._render创建一个VNode。

> vm._render 最终是通过执⾏ createElement ⽅法并返回的是 vnode

1.2 VNode 渲染成真实的DOM

这个是通过vm._update完成的。方法定义在core/instance/lifecycle.js中

\_update核心是调用\_\_patch_方法。

```
if (!prevVnode) {
      // initial render
      vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
    } else {
      // updates
      vm.$el = vm.__patch__(prevVnode, vnode)
    }
```

在web平台的调用在platforms/web/runtime/index.js中，共同的核心逻辑在core/vdom/patch.js因为在服务器端和平台端的不一样，这里vue采用curry化patch核心代码。避免重复传入nodeOps和modules，核心patch代码接收4个参数oldVnode（旧的vnode节点）, vnode（执行_render返回的vnode节点）, hydrating（表示服务器端）, removeOnly（是给 transition-group ⽤的）。

>  vm._update 的⽅法⾥是这么调⽤ patch ⽅法的：

```
// initial render 
vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
```

vm.$el 的赋值是在之前 mountComponent 函数做的， vnode 对应的是调⽤ render 函数的返回值， hydrating 在⾮服务端渲染情况下为 false， removeOnly 为 false。

初始化传入的app是一个真实的dom节点，进入isRealElement流程，

```
oldVnode = emptyNodeAt(oldVnode)
```

通过 emptyNodeAt ⽅法把 oldVnode 转换成 VNode 对象，然后再调⽤ createElm ⽅法。**createElm** 的作⽤是通过虚拟节点创建真实的 DOM 并插⼊到它的⽗节点中。

> 最后调⽤ insert ⽅法把 DOM 插⼊到⽗节点中，因为是递归调⽤，⼦元素会优先调⽤ insert ， 
>
> 所以整个 vnode 树节点的插⼊顺序是先⼦后⽗。

> 再回到 patch ⽅法，⾸次渲染我们调⽤了 createElm ⽅法，这⾥传⼊的 parentElm 是 oldVnode.elm 的⽗元素， 在我们的例⼦是 id 为 #app div 的⽗元素，也就是 Body；实际上整个过 程就是递归创建了⼀个完整的 DOM 树并插⼊到 Body 上。

> 最后，我们根据之前递归 createElm ⽣成的 vnode 插⼊顺序队列，执⾏相关的 insert 钩⼦函数，

初始化Vue到最终渲染的整个过程：

new Vue --> init --> $mount --> compiler --> render --> vnode --> patch -->DOM

按照个人理解重新理一下流程：

a、new Vue这里实例化Vue，调用Vue核心方法中的this._init方法

b、init方法中使用$mount挂载（vm.$options.el）el:'#App'

c、之后进行编译，是否是template，如果是编译成render函数可以的

d、render函数，如果是entery-runtime的话就不会进入template直接进行render方法，将元素转换成vnode

e、vnode要经过create，diff，patch等方法才能将vnode转换成真实dom

其实这个vnode就是模拟版dom树，vue的方法是通过insert插入进去（node.appendChild）。在初始化时，就是递归创建了一个完整的DOM树并插入到Body上。

之后跳过组件化学习，先学习深入响应式原理