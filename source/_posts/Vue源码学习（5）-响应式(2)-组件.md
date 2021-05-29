---
title: Vue源码学习（5）响应式-组件
date: 2021-05-25 10:37:14
tags: vue
cover: https://raw.githubusercontent.com/DaMu2018/cloudimg/main/data/vue.png
top_img: https://raw.githubusercontent.com/DaMu2018/cloudimg/main/data/ready-to-go.jpg
---

*最近太忙了没有保持更新😂，之前的顺序发错了，补发一下，5响应式补充-组件*

### 组件更新

```js
updateComponent = () => {
	vm._update(vm._render(), hydrating)
}
new Watcher(vm, updateComponent, noop, {
    before () {
        if (vm._isMounted && !vm._isDestroyed) {
            callHook(vm, 'beforeUpdate')
        }
    }
}, true /* isRenderWatcher */)
```

`vm._update`触发函数`vm.$el = vm.__patch__(prevVnode, vnode)`,调用`patch`函数`src/core/vdom/patch.js`,更新组件有几种策略。

- 新旧节点不同

  不同时，创建新节点 -> 更新占位节点 -> 删除旧节点

- 新旧节点相同

  调用`patchVnode`

  - 执行prepatch

    ```js
    let i
    const data = vnode.data
    if (isDef(data) && isDef(i = data.hook) && isDef(i = 		i.prepatch)) {
    	i(oldVnode, vnode)
    }
    // ------------
    // create-component.js
    prepatch (oldVnode: MountedComponentVNode, vnode: MountedComponentVNode) {
        const options = vnode.componentOptions
        const child = vnode.componentInstance = oldVnode.componentInstance
        updateChildComponent(
          child,
          options.propsData, // updated props
          options.listeners, // updated listeners
          vnode, // new parent vnode
          options.children // new children
        )
      }
    // ------------
    // liftcycle.js
    function updateChildComponent{
    //....
    	vm.$options._parentVnode = parentVnode
      	vm.$vnode = parentVnode // update vm's placeholder node without re-render
    
      	if (vm._vnode) { // update child tree's parent
       	 vm._vnode.parent = parentVnode
      	}
      	vm.$options._renderChildren = renderChildren
    
      // update $attrs and $listeners hash
      // these are also reactive so they may trigger child update if the child
      // used them during render
      	vm.$attrs = parentVnode.data.attrs || emptyObject
      	vm.$listeners = listeners || emptyObject    
        // .......
    }
         
    ```

    updateChildComponent更新了vnode，vnode实例对应的vm的属性都发生变化，包括占位符$vnode,slot,listeners,props的更新等。

  - 执行update钩子函数

    如果是文本节点，直接替换文本内容，如果不是分情况处理。

  - 执行`postpatch`钩子函数

    ```js
    if (isDef(data)) {
          if (isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode, vnode)
        }
    ```

    

#### updateChildren

patch的核心方法，diff算法的实现。oldStartIndex和newStartIndex，依次按顺序对比。

#### props

在初始化props前，会将props格式化，确保props都是对象格式。

```js
/**
 * Ensure all props option syntax are normalized into the
 * Object-based format.
 */
function normalizeProps (options: Object, vm: ?Component)
```

props有两种写法：

```js
if (Array.isArray(props)) {
    i = props.length
    while (i--) {
      val = props[i]
      if (typeof val === 'string') {
        name = camelize(val)
        res[name] = { type: null }
      } else if (process.env.NODE_ENV !== 'production') {
        warn('props must be strings when using array syntax.')
      }
    }
  } else if (isPlainObject(props)) {
    for (const key in props) {
      val = props[key]
      name = camelize(key)
      res[name] = isPlainObject(val)
        ? val
        : { type: val }
    }
  }
```

一种是array，但是props里面的必须是string类型的，一种是对象，把属性取出来。

转成驼峰格式

```js
/**
 * Camelize a hyphen-delimited string.
 */
const camelizeRE = /-(\w)/g
export const camelize = cached((str: string): string => {
  return str.replace(camelizeRE, (_, c) => c ? c.toUpperCase() : '')
})
```

props主要做了：校验、响应式、代理

校验：校验props的传值是否类型正确

响应：父组件传值改变时，更新子组件

代理：this._props.xxx = this.xxx

