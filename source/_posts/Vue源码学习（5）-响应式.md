---
title: Vue源码学习（5）-响应式
date: 2021-5-7 09:26:07
tags: vue
cover: https://raw.githubusercontent.com/DaMu2018/cloudimg/main/data/vue.png
top_img: https://raw.githubusercontent.com/DaMu2018/cloudimg/main/data/ready-to-go.jpg
---

### 响应式原理

模板和数组渲染成DOM过程：

new Vue --> init -->$mount -->`mountComponent `--> vm.\_render--> `createElement`或者`createComponent`  -->vnode --> vm._update --> `vm.__patch__` -->`createElm`（将vnode转为DOM插入到父节点中）

Vue的一个特性是数据驱动渲染DOM，另一个就是数据变更触发DOM变化。

- 响应式对象

  Object.defineProperty

  Vue初始化时会触发`_init`执行`initState(vm)`

  ```js
  export function initState (vm: Component) {
    vm._watchers = []
    const opts = vm.$options
    if (opts.props) initProps(vm, opts.props)
    if (opts.methods) initMethods(vm, opts.methods)
    if (opts.data) {
      initData(vm)
    } else {
      observe(vm._data = {}, true /* asRootData */)
    }
    if (opts.computed) initComputed(vm, opts.computed)
    if (opts.watch && opts.watch !== nativeWatch) {
      initWatch(vm, opts.watch)
    }
  }
  ```

  - initProps

    调用`defineReactive(props, key, value)`，把prop对应的值改成响应式。

    ```js
    if (!(key in vm)) {
          proxy(vm, `_props`, key)
    }
    ```

    vm.\_props.xxx访问对应属性，通过proxy可以使用vm.xxx访问props属性。

  - initData

    ```js
    observe(data, true /* asRootData */)
    ```

  - Observe

    ```js
    /**
     * Attempt to create an observer instance for a value,
     * returns the new observer if successfully observed,
     * or the existing observer if the value already has one.
     */
    export function observe (value: any, asRootData: ?boolean): Observer | void {
      if (!isObject(value) || value instanceof VNode) {
        return
      }
      let ob: Observer | void
      if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
          //如果原来有observer对象，返回当前的数据
          ob = value.__ob__
      } else if (
        shouldObserve &&
        !isServerRendering() &&
        (Array.isArray(value) || isPlainObject(value)) &&
        Object.isExtensible(value) &&
        !value._isVue
      ) {
         //没有则新建一个Ovserver对象
        ob = new Observer(value)
      }
      if (asRootData && ob) {
        ob.vmCount++
      }
      return ob
    }
    ```

    `Observer`是一个对象

    ```js
    /**
     * Observer class that is attached to each observed
     * object. Once attached, the observer converts the target
     * object's property keys into getter/setters that
     * collect dependencies and dispatch updates.
     */
    export class Observer {
      value: any;
      dep: Dep;
      vmCount: number; // number of vms that have this object as root $data
    
      constructor (value: any) {
        this.value = value
        this.dep = new Dep()
        this.vmCount = 0
        //def是Object.defineProperty的封装
        def(value, '__ob__', this)
         //处理数组
        if (Array.isArray(value)) {
          if (hasProto) {
            //原型增强
            protoAugment(value, arrayMethods)
          } else {
            copyAugment(value, arrayMethods, arrayKeys)
          }
          this.observeArray(value)
        } else {
          this.walk(value)
        }
      }
    
      /**
       * Walk through all properties and convert them into
       * getter/setters. This method should only be called when
       * value type is Object.
       */
      walk (obj: Object) {
        const keys = Object.keys(obj)
        for (let i = 0; i < keys.length; i++) {
          //将数据设置为响应式
          defineReactive(obj, keys[i])
        }
      }
    
      /**
       * Observe a list of Array items.
       */
      observeArray (items: Array<any>) {
        for (let i = 0, l = items.length; i < l; i++) {
          observe(items[i])
        }
      }
    }
    ```

    `defineReactive`核心方法：

    ```js
    Object.defineProperty(obj, key, {
        enumerable: true,
        configurable: true,
        get: function reactiveGetter () {
          const value = getter ? getter.call(obj) : val
          if (Dep.target) {
            //依赖收集
            dep.depend()
            if (childOb) {
              childOb.dep.depend()
              if (Array.isArray(value)) {
                dependArray(value)
              }
            }
          }
          return value
        },
        set: function reactiveSetter (newVal) {
          const value = getter ? getter.call(obj) : val
          /* eslint-disable no-self-compare */
          if (newVal === value || (newVal !== newVal && value !== value)) {
            return
          }
          /* eslint-enable no-self-compare */
          if (process.env.NODE_ENV !== 'production' && customSetter) {
            customSetter()
          }
          // #7981: for accessor properties without setter
          if (getter && !setter) return
          if (setter) {
            setter.call(obj, newVal)
          } else {
            val = newVal
          }
          childOb = !shallow && observe(newVal)
          //依赖通知
          dep.notify()
        }
      })
    ```

    

### 依赖收集

```js
//...
const dep = new Dep()
//...
get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        //依赖收集
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    },
//...
```

```js
export default class Dep {
  constructor() {
    this.id = uid++;
    this.subs = [];
  }
  //...
  addSub(sub: Watcher) {
    this.subs.push(sub);
  }
  //...
  depend() {
    if (Dep.target) {
      Dep.target.addDep(this);
    }
  }
  notify() {
    // stabilize the subscriber list first
    const subs = this.subs.slice();
    if (process.env.NODE_ENV !== "production" && !config.async) {
      // subs aren't sorted in scheduler if not running async
      // we need to sort them now to make sure they fire in correct
      // order
      subs.sort((a, b) => a.id - b.id);
    }
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update();
    }
  }
  //...
}
```

> `Dep` 实际上就是对 `Watcher` 的一种管理，`Dep` 脱离 `Watcher` 单独存在是没有意义的

 Vue 的 mount 过程是通过 `mountComponent`，其中

```js
new Watcher(vm, updateComponent, noop, {
  before () {
    if (vm._isMounted) {
      callHook(vm, 'beforeUpdate')
    }
  }
}, true /* isRenderWatcher */)
```

Watcher初始化时会触发pushTarget(this)

```js
export function pushTarget (_target: Watcher) {
  if (Dep.target) targetStack.push(Dep.target)
  Dep.target = _target
}
```

`value = this.getter.call(vm, vm)`这个对应`updateComponent`函数

触发vm.\_update(vm.\_render(),hydrating),`_render()`会生成vnode,这个时候会对`vm`

上的数据访问,这个时候会触发数据对象的getter。

> 每个对象值得getter都会持有一个`dep`,触发getter的时候会调用`dep.depend()`方法，会执行Dep.target.addDep(this)

dep.addDep会执行将数据添加（保证统一数据不会重复添加），把当前watcher订阅到这个数据持有的`dep`的`subs`中，目的是为了后续数据变化通知别的subs做准备。

```js
/**
   * Evaluate the getter, and re-collect dependencies.
   */
  get () {
    pushTarget(this)
    let value
    const vm = this.vm
    try {
      value = this.getter.call(vm, vm)
    } catch (e) {
      if (this.user) {
        handleError(e, vm, `getter for watcher "${this.expression}"`)
      } else {
        throw e
      }
    } finally {
      // "touch" every property so they are all tracked as
      // dependencies for deep watching
      if (this.deep) {
        traverse(value)
      }
      popTarget()
      this.cleanupDeps()
    }
    return value
  }

```

依赖收集后会对依赖进行清空`this.cleanupDeps()`。

> Vue是数据驱动的，每次数据变化都会重新render，vm._render会再次执行，并触发数据的getters。

Vue每次添加完新的订阅，会移除旧的订阅。避免多余的操作影响性能。

### 派发更新

派发更新是在set，修改数据时触发，去修改dom元素。

```js
set: function reactiveSetter (newVal) {
      const value = getter ? getter.call(obj) : val
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter()
      }
      // #7981: for accessor properties without setter
      if (getter && !setter) return
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      childOb = !shallow && observe(newVal)
      dep.notify()
    }
```

dep.notify()通知订阅者

```js
notify () {
    // stabilize the subscriber list first
    const subs = this.subs.slice()
    if (process.env.NODE_ENV !== 'production' && !config.async) {
      // subs aren't sorted in scheduler if not running async
      // we need to sort them now to make sure they fire in correct
      // order
      subs.sort((a, b) => a.id - b.id)
    }
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
```

调用watcher的update

```js
update () {
    /* istanbul ignore else */
    if (this.lazy) {
      this.dirty = true
    } else if (this.sync) {
      this.run()
    } else {
      queueWatcher(this)
    }
  }
```

queueWatcher就是把watcher添加到一个队列里。Vue做派发更新的时候并不会每次数据改变会触发watcher的回调。在队列一定程度的时候使用`nextTick`后执行，`flushSchedulerQueue`

flushSchedulerQueue中`queue.sort((a, b) => a.id - b.id)`

- 队列排序，

  1.组件更新由父到子，watcher的创建也是由父到子，执行顺序也是保持先父后子。2.用户自定义的watcher要优先于watcher执行

  3.组件在父组件watcher执行期间被销毁，它对应的watcher都可以被跳过。

- 队列遍历

  watcher.run()

  ```js
    /**
     * Scheduler job interface.
     * Will be called by the scheduler.
     */
    run () {
      if (this.active) {
        const value = this.get()
        if (
          value !== this.value ||
          // Deep watchers and watchers on Object/Arrays should fire even
          // when the value is the same, because the value may
          // have mutated.
          isObject(value) ||
          this.deep
        ) {
          // set new value
          const oldValue = this.value
          this.value = value
          if (this.user) {
            try {
              this.cb.call(this.vm, value, oldValue)
            } catch (e) {
              handleError(e, this.vm, `callback for watcher "${this.expression}"`)
            }
          } else {
            this.cb.call(this.vm, value, oldValue)
          }
        }
      }
    }
  ```

  

