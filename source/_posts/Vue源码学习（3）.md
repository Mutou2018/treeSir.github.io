---
title: Vue源码学习（3）
date: 2021-4-25 18:46:42
tags: vue
cover: https://raw.githubusercontent.com/DaMu2018/cloudimg/main/data/vue.png
top_img: https://raw.githubusercontent.com/DaMu2018/cloudimg/main/data/ready-to-go.jpg
---
Vue源码学习（3）

> vue技术揭秘笔记

### 组件

组件创建主要是基于core/vdom/create-component.js。

```js
function createComponent (

 Ctor: Class<Component> | Function | Object | void,

 data: ?VNodeData,

 context: Component,

 children: ?Array<VNode>,

 tag?: string

){}
```

核心步骤主要有三个：

- 构造子类构造函数

  ```js
  const baseCtor = context.$options._base
  
    // plain options object: turn it into a constructor
    if (isObject(Ctor)) {
      Ctor = baseCtor.extend(Ctor)
    }
  ```

  vue创建组件的时候

  ```js
  import HelloWorld from './components/HelloWorld'
  export default {
    name: 'app',
    components: {
      HelloWorld
    }
  }
  ```

  `vm.$options._base` 拿到 Vue 这个构造函数了。Vue.extend函数的作用就是构造Vue的子类。

  ```js
  const Sub = function VueComponent (options) {
     this._init(options)
  }
  ```

- 安装组件钩子

  ```js
  // install component management hooks onto the placeholder node
  installComponentHooks(data)
  ```

- 实例化 `vnode`

  ```js
  const name = Ctor.options.name || tag
  const vnode = new VNode(
    `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
    data, undefined, undefined, undefined, context,
    { Ctor, propsData, listeners, tag, children },
    asyncFactory
  )
  return vnode
  ```

createComponent要返回一个组件的vnode。

### patch

createComponent会使用\_\_path\_\_ ,在`vdom/patch.js`中使用createElm方法。在子组件挂载时触发：

```js
  function createComponent (vnode, insertedVnodeQueue, parentElm, refElm) {
    let i = vnode.data
    if (isDef(i)) {
      const isReactivated = isDef(vnode.componentInstance) && i.keepAlive
      if (isDef(i = i.hook) && isDef(i = i.init)) {
        i(vnode, false /* hydrating */)
      }
      // after calling the init hook, if the vnode is a child component
      // it should've created a child instance and mounted it. the child
      // component also has set the placeholder vnode's elm.
      // in that case we can just return the element and be done.
      if (isDef(vnode.componentInstance)) {
        initComponent(vnode, insertedVnodeQueue)
        insert(parentElm, vnode.elm, refElm)
        if (isTrue(isReactivated)) {
          reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm)
        }
        return true
      }
    }
  }
```

组件初始化时会调用组件init函数，普通非keep-alive的组件是挂在vnode.componentInstance上的，`createComponentInstanceForVnode(vnode,activeInstance)`,这里会返回new vnode.componentOptions.Ctor(options)，之前组件Ctor就是新实例化Vue，重新走一遍_init，但是因为是组件所以触发的函数是`initInternalComponent`,组件`child.$mount(hydrating ? vnode.elm : undefined, hydrating)`初始化后$mount，执行mountComponent，挂载组件，再次回到之前的$mount->render->vnode->patch。在`vm_render`生成vnode之后，之后就要执行 `vm. _update` 。

> Vue的初始化是一个深度遍历的过程，在实例化子组件的过程中,需要知道当前的上下文的Vue实例是什么，并把它作为子组件的父Vue实例。

```js
initComponent(vnode, insertedVnodeQueue)
insert(parentElm, vnode.elm, refElm)
```

完成组件initComponent过程后，执行insert完成dom插入，所以组件DOM的插入顺序是先子后父。

无论是组件还是项目初始都会执行_init(options),而\_init会执行mergeOptions

```js
	// merge options
    if (options && options._isComponent) {
      initInternalComponent(vm, options)
    } else {
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }
```

项目初始化时的Vue.options是在`initGlobalAPI(Vue)`中定义了这个值。

```js
//shared/constants.js
ASSET_TYPES = [
  'component',
  'directive',
  'filter'
]
// global-api/index.js
ASSET_TYPES.forEach(type => {
    Vue.options[type + 's'] = Object.create(null)
})
```

`mergeOptions `的合并返回到options，最终获得的options如下：

```js
vm.$options = mergeOptions(
  resolveConstructorOptions(vm.constructor),
  options || {},
  vm
)
vm.$options = {
  components: { },
  created: [
    function created() {
      console.log('parent created')
    }
  ],
  directives: { },
  filters: { },
  _base: function Vue(options) {
    // ...
  },
  el: "#app",
  render: function (h) {
    //...
  }
}
```

组件场景的返回的option

```js
vm.$options = {
  parent: Vue /*父Vue实例*/,
  propsData: undefined,
  _componentTag: undefined,
  _parentVnode: VNode /*父VNode实例*/,
  _renderChildren:undefined,
  __proto__: {
    components: { },
    directives: { },
    filters: { },
    _base: function Vue(options) {
        //...
    },
    _Ctor: {},
    created: [
      function created() {
        console.log('parent created')
      }, function created() {
        console.log('child created')
      }
    ],
    mounted: [
      function mounted() {
        console.log('child mounted')
      }
    ],
    data() {
       return {
         msg: 'Hello Vue'
       }
    },
    template: '<div>{{msg}}</div>'
  }
}
```

> 对于 `options` 的合并有 2 种方式
>
> - 子组件初始化过程通过 `initInternalComponent` 方式
> - 外部初始化 Vue 通过 `mergeOptions` 的方式，合并完的结果保留在 `vm.$options` 中。

组件的初始化比外部的`mergeOptions` 要快。