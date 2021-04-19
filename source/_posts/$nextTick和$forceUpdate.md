---
title: $nextTick和$forceUpdate解析
date: 2021-04-19 09:14:48
tags: vue
cover: https://raw.githubusercontent.com/DaMu2018/cloudimg/main/data/vue.png
top_img: https://raw.githubusercontent.com/DaMu2018/cloudimg/main/data/ready-to-go.jpg
description: $nextTick和$forceUpdate解析
---
# $nextTick和$forceUpdate

​	今天遇到个问题,感觉之前也遇到过,但是今天想去研究下,为什么会出现这种情况。

​	bug：

​			element的select组件切换时，data里面的对象值已经改变，但是页面上没有响应。

​	解决思路：

​			开始以为是没有改成功值的原因，就想着使用$set，来修改值，结果并没有用，然后就watch监听，发现值已经修改了，但是视图没有刷新，网上查了下资料发现是因为嵌套太深了，可能导致视图更新不及时。可以使用$forceUpdate。不过笔者想要研究研究forceUpdate为什么会使渲染生效。

​	$forceUpdate:

​	官网上的说法：forceUpdate，在vue中做强制更新。$forceUpdate迫使vue实例重新渲染。它仅仅影响实例本身和插槽内容的子组件，而不是所有的子组件。

​	$forceUpdate只会触发beforeUpdate和updated这两个生命周期。因为它是迫使vue实例重新渲染，所以会重新触发render函数渲染虚拟dom。

​	vue中源码

```vue
Vue.prototype.$forceUpdate = function () {
    const vm: Component = this
    if (vm._watcher) {
      vm._watcher.update()
    }
  }
```

_watcher.update()就是触发重新渲染。

所以出现这个bug的原因是因为修改v-model的值时，render函数没有触发，需要强制更新。

​	$nextTick()

​	将回调延迟到下次Dom更新循环之后，在修改数据后立即使用它，然后等待dom更新。

​	根据vue中的源码来看。

```
//render.js
Vue.prototype.$nextTick = function (fn: Function) {
    return nextTick(fn, this)
}
// next-tick.js
// Here we have async deferring wrappers using microtasks.
// In 2.5 we used (macro) tasks (in combination with microtasks).
// However, it has subtle problems when state is changed right before repaint
// So we now use microtasks everywhere, again.

let timerFunc
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve()
  timerFunc = () => {
    p.then(flushCallbacks)
    // In problematic UIWebViews, Promise.then doesn't completely break, but
    // it can get stuck in a weird state where callbacks are pushed into the
    // microtask queue but the queue isn't being flushed, until the browser
    // needs to do some other work, e.g. handle a timer. Therefore we can
    // "force" the microtask queue to be flushed by adding an empty timer.
    if (isIOS) setTimeout(noop)
  }
  isUsingMicroTask = true
}
```

源码主要使用promise优先，但是因为在IOS >= 9.3.3版本会有问题，所以当是IOS平台时，使用setTimeout。

next-tick.js对外暴露了nextTick这一个参数，每次调用nextTick时执行：

```
export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  if (!pending) {
    pending = true
    timerFunc()
  }
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}

```



- 如果有回调函数，将回调函数push进入callback数组

  ```
  callbacks.push(() => {
      if (cb) {
        try {
          cb.call(ctx)
        } catch (e) {
          handleError(e, ctx, 'nextTick')
        }
      } else if (_resolve) {
        _resolve(ctx)
      }
    })
  ```

- 然后执行timeFunc函数，延迟调用flushCallbacks函数

- 会遍历执行所有回调函数

  这么比较下来感觉，$forceUpdate()和$nextTick()这两个方法。nextTick是将dom渲染之后可以获取更新后的dom，而forceUpdate则是因为改变值之后render没有触发，有forceUpdate来强制触发render方法。

