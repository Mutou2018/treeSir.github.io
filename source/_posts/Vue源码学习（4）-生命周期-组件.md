---
title: Vue源码学习（4）
date: 2021年4月30日 10:17:03
tags: vue
cover: https://raw.githubusercontent.com/DaMu2018/cloudimg/main/data/vue.png
top_img: https://raw.githubusercontent.com/DaMu2018/cloudimg/main/data/ready-to-go.jpg
---

#### 生命周期 

​		Vue实例创建时，会进行很多的初始设置。比如设置数据监听、编译模板、挂载实例到DOM，在数据变化时更新DOM。这里会有生命周期的钩子函数，可以让我们在一些场景下添加业务代码。

​		生命周期函数都是调用`callHook`方法，都是对应`vm.$option[hook]`对应的回调函数数组，然后遍历执行，执行的时候把`vm`作为函数的执行上下文。

> 各个阶段的生命周期的函数也被合并到 `vm.$options` 里，并且是一个数组。

- beforeCreate&created

  beforeCreate和created函数都是在实例化`vue`的阶段，在_init执行的。

  ```js
  Vue.prototype._init = function (options?: Object) {
  	...
  	initLifecycle(vm)
      initEvents(vm)
      initRender(vm)
      callHook(vm, 'beforeCreate')
      initInjections(vm) // resolve injections before data/props
      initState(vm)
      initProvide(vm) // resolve provide after data/props
      callHook(vm, 'created')
      ...
  }
  ```

  initState的作用是初始化`data`,`props`,`watch`,`computed`等属性。在`created`是可以执行data等属性的访问，在beforeCreated时间不能访问这些属性。

- beforeMounted&mounted

  beforeMounted钩子函数发生在`mount`，在DOM挂载之前，调用时间就是在`mountComponent`函数中。在执行`vm._render`之前执行了beforeUpdate钩子函数。在`vm._update`把VNode patch到真实DOM后，执行mounted钩子。

  ```js
  if (vm.$vnode == null) { //不是一次组件的初始化过程，是new vue初始化过程
      vm._isMounted = true
      callHook(vm, 'mounted')
    }
  ```

  组件mounted，是vnode patch到DOM后，会执行`invokeInsertHook`函数，把`insertedVnodeQueue`里面保存的钩子函数依次执行一遍。钩子函数定义在：`src/core/vdom/create-component.js`

  ```js
  const componentVNodeHooks = {
    // ...
    insert (vnode: MountedComponentVNode) {
      const { context, componentInstance } = vnode
      if (!componentInstance._isMounted) {
        componentInstance._isMounted = true
        callHook(componentInstance, 'mounted')
      }
      // ...
    },
  }
  ```

  ``insertedVnodeQueue`的添加顺序是先子后父，对于同步渲染的子组件而言，`mounted`钩子函数的执行顺序也是先子后父。

- beforeUpdate&updated

  beforeUpdate和updated的钩子函数的执行时间都是在数据更新的时候。

  beforeIUpdate执行时机都是在渲染watcher的before函数中。

  updated的执行时机是在`flushSchedulerQueue`函数调用的时候。

- beforeDestroy&destoryed

  在组件销毁阶段执行，销毁组件时所做的处理：

  ```js
  //删除自身
  if (parent && !parent._isBeingDestroyed && !vm.$options.abstract) {
  	remove(parent.$children, vm)
  }
  //删除watcher
  if (vm._watcher) {
  	vm._watcher.teardown()
  }
  //
  // remove reference from data ob
  // frozen object may not have observer.
  if (vm._data.__ob__) {
  	vm._data.__ob__.vmCount--
  }
  // call the last hook...
  vm._isDestroyed = true
  // invoke destroy hooks on current rendered tree
  vm.__patch__(vm._vnode, null)
  ```

  > 在 `$destroy` 的执行过程中，它又会执行 `vm.__patch__(vm._vnode, null)` 触发它子组件的销毁钩子函数，这样一层层的递归调用，所以 `destroy` 钩子函数执行顺序是先子后父，和 `mounted` 过程一样。

#### 组件注册

​		组件注册分成2种，全局注册和局部注册。

- 全局注册

  ```js
  export const ASSET_TYPES = [
    'component',
    'directive',
    'filter'
  ]
  // 初始默认方法
  export function initAssetRegisters (Vue: GlobalAPI) {
    /**
     * Create asset registration methods.
     */
    ASSET_TYPES.forEach(type => {
      Vue[type] = function (
        id: string,
        definition: Function | Object
      ): Function | Object | void {
        if (!definition) {
          return this.options[type + 's'][id]
        } else {
          /* istanbul ignore if */
          if (process.env.NODE_ENV !== 'production' && type === 'component') {
            validateComponentName(id)
          }
          if (type === 'component' && isPlainObject(definition)) {
            definition.name = definition.name || id
            definition = this.options._base.extend(definition)
          }
          if (type === 'directive' && typeof definition === 'function') {
            definition = { bind: definition, update: definition }
          }
          this.options[type + 's'][id] = definition
          return definition
        }
      }
    })
  }
  ```

  this.options._base.extend(definition)这个是把对象转成继承于Vue的构造函数

- 局部注册

  组件Vue的实例化阶段有一个合并option的逻辑。在`resolveAsset`拿到组件的构造函数，作为`createComponent`的钩子的参数。

  区别就在于一个是挂载在Vue.options上的一个是挂载在vm.$options.components。

### 异步组件

用法1. 普通异步

```js
Vue.component('async-example', function (resolve, reject) {
   // 这个特殊的 require 语法告诉 webpack
   // 自动将编译后的代码分割成不同的块，
   // 这些块将通过 Ajax 请求自动下载。
   require(['./my-async-component'], resolve)
})
```

用法2.promise

```js
Vue.component(
  'async-webpack-example',
  // 该 `import` 函数返回一个 `Promise` 对象。
  () => import('./my-async-component')
)
```

用法3.

```js
const AsyncComp = () => ({
  // 需要加载的组件。应当是一个 Promise
  component: import('./MyComp.vue'),
  // 加载中应当渲染的组件
  loading: LoadingComp,
  // 出错时渲染的组件
  error: ErrorComp,
  // 渲染加载中组件前的等待时间。默认：200ms。
  delay: 200,
  // 最长等待时间。超出此时间则渲染错误组件。默认：Infinity
  timeout: 3000
})
Vue.component('async-example', AsyncComp)
```

使用`resolveAsyncComponent`函数vdom/helper/resolve-async-component.js中，进行异步组件的使用。异步函数需要保证只加载一次，使用闭包和标志位保证包装的函数只会执行一次。

```js
// src/shared/util.js
/**
 * Ensure a function is called only once.
 */
export function once (fn: Function): Function {
  let called = false
  return function () {
    if (!called) {
      called = true
      fn.apply(this, arguments)
    }
  }
}
```

`const res = factory(resolve, reject)`中，如果加载成功则调用$forceUpdate强制组件重新渲染一次。

promise结合webpack异步加载的语法糖`()=>import('./my-async-component')`

自定义设置的异步中又包含了另一个异步。

```js
const res = factory(resolve, reject)

    if (isObject(res)) {
        //处理promise语法
      if (isPromise(res)) {
        // () => Promise
        if (isUndef(factory.resolved)) {
          res.then(resolve, reject)
        }
      } else if (isPromise(res.component)) {
        res.component.then(resolve, reject)

        if (isDef(res.error)) {
          factory.errorComp = ensureCtor(res.error, baseCtor)
        }

        if (isDef(res.loading)) {
          factory.loadingComp = ensureCtor(res.loading, baseCtor)
          if (res.delay === 0) {
            factory.loading = true
          } else {
            timerLoading = setTimeout(() => {
              timerLoading = null
              if (isUndef(factory.resolved) && isUndef(factory.error)) {
                factory.loading = true
                forceRender(false)
              }
            }, res.delay || 200)
          }
        }

        if (isDef(res.timeout)) {
          timerTimeout = setTimeout(() => {
            timerTimeout = null
            if (isUndef(factory.resolved)) {
              reject(
                process.env.NODE_ENV !== 'production'
                  ? `timeout (${res.timeout}ms)`
                  : null
              )
            }
          }, res.timeout)
        }
      }
    }
```

异步组件会创建一个注释节点作为占位符。

```js
Ctor = resolveAsyncComponent(asyncFactory, baseCtor)
	//如果不是loading组件，一般返回的都是undefined
    if (Ctor === undefined) {
      // return a placeholder node for async component, which is rendered
      // as a comment node but preserves all the raw information for the node.
      // the information will be used for async server-rendering and hydration.
      return createAsyncPlaceholder(
        asyncFactory,
        data,
        context,
        children,
        tag
      )
    }
```

> 异步组件实现的本质是 2 次渲染，除了 0 delay 的高级异步组件第一次直接渲染成 loading 组件外，其它都是第一次渲染生成一个注释节点，当异步获取组件成功后，再通过 `forceRender` 强制重新渲染，这样就能正确渲染出我们异步加载的组件了。

