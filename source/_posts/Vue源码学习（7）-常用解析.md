---
title: Vue源码学习（7）-常用解析
date: 2021-05-29 12:31:54
tags: vue
cover: https://raw.githubusercontent.com/DaMu2018/cloudimg/main/data/vue.png
top_img: https://raw.githubusercontent.com/DaMu2018/cloudimg/main/data/ready-to-go.jpg
---

### v-model

> 数据响应式数据改变去驱动DOM视图变化，双向绑定是除开数据驱动DOM外，DOM变化反过来影响数据。

`model`的定义在`src/platforms/web/compiler/directives/model.js`

```js
if (el.component) {
    genComponentModel(el, value, modifiers)
    // component v-model doesn't need extra runtime
    return false
  } else if (tag === 'select') {
    genSelect(el, value, modifiers)
  } else if (tag === 'input' && type === 'checkbox') {
    genCheckboxModel(el, value, modifiers)
  } else if (tag === 'input' && type === 'radio') {
    genRadioModel(el, value, modifiers)
  } else if (tag === 'input' || tag === 'textarea') {
    genDefaultModel(el, value, modifiers)
  } else if (!config.isReservedTag(tag)) {
    genComponentModel(el, value, modifiers)
    // component v-model doesn't need extra runtime
    return false
  }
```

demo:

`<input v-model='message'/>`，在`genAssignmentCode`中

```js
export function genAssignmentCode (
  value: string,
  assignment: string
): string {
  const res = parseModel(value)
  if (res.key === null) {
    return `${value}=${assignment}`
  } else {
    return `$set(${res.exp}, ${res.key}, ${assignment})`
  }
}
```

v-model对应的value做解析，value对应`message=$event.target.value`。

```js
function genDefaultModel (
  el: ASTElement,
  value: string,
  modifiers: ?ASTModifiers
): ?boolean {
  const type = el.attrsMap.type

  // warn if v-bind:value conflicts with v-model
  // except for inputs with v-bind:type
  if (process.env.NODE_ENV !== 'production') {
    const value = el.attrsMap['v-bind:value'] || el.attrsMap[':value']
    const typeBinding = el.attrsMap['v-bind:type'] || el.attrsMap[':type']
    if (value && !typeBinding) {
      const binding = el.attrsMap['v-bind:value'] ? 'v-bind:value' : ':value'
      warn(
        `${binding}="${value}" conflicts with v-model on the same element ` +
        'because the latter already expands to a value binding internally',
        el.rawAttrsMap[binding]
      )
    }
  }

  const { lazy, number, trim } = modifiers || {}
  const needCompositionGuard = !lazy && type !== 'range'
  const event = lazy
    ? 'change'
    : type === 'range'
      ? RANGE_TOKEN
      : 'input'

  let valueExpression = '$event.target.value'
  if (trim) {
    valueExpression = `$event.target.value.trim()`
  }
  if (number) {
    valueExpression = `_n(${valueExpression})`
  }

  let code = genAssignmentCode(value, valueExpression)
  if (needCompositionGuard) {
    code = `if($event.target.composing)return;${code}`
  }

  addProp(el, 'value', `(${value})`)
  addHandler(el, event, code, null, true)
  if (trim || number) {
    addHandler(el, 'blur', '$forceUpdate()')
  }
}
```

函数做`addProp`和`addHandler`作用是在ast中添加事件。转换成：

```js
<input
  v-bind:value="message"
  v-on:input="message=$event.target.value">
```

- 组件v-model

```js
let Child = {
  template: '<div>'
  + '<input :value="value" @input="updateValue" placeholder="edit me">' +
  '</div>',
  props: ['value'],
  methods: {
    updateValue(e) {
      this.$emit('input', e.target.value)
    }
  }
}

let vm = new Vue({
  el: '#app',
  template: '<div>' +
  '<child v-model="message"></child>' +
  '<p>Message is: {{ message }}</p>' +
  '</div>',
  data() {
    return {
      message: ''
    }
  },
  components: {
    Child
  }
})
```

父组件在编译阶段会解析`v-model`指令，执行`genData`中的`genDirective`函数之后执行`src/platforms/web/compiler/directives/model.js` 中定义的 `model` 函数，之后执行`genComponentModel` 函数定义在 `src/compiler/directives/model.js` 中。

```js
el.model = {
  callback:'function ($$v) {message=$$v}',
  expression:'"message"',
  value:'(message)'
}
```

父组件最终生成的render：

```js
with(this){
  return _c('div',[_c('child',{
    model:{
      value:(message),
      callback:function ($$v) {
        message=$$v
      },
      expression:"message"
    }
  }),
  _c('p',[_v("Message is: "+_s(message))])],1)
}
```

创建组组件的vnode阶段，执行createComponent函数，`src/core/vdom/create-component.js` 中：

```js
// transform component v-model data into props & events
export function createComponent (){
   if (isDef(data.model)) {
     transformModel(Ctor.options, data)
   }
// ...
}
// transform component v-model info (value and callback) into
// prop and event handler respectively.
function transformModel (options, data: any) {
  const prop = (options.model && options.model.prop) || 'value'
  //可以配置v-model
  const event = (options.model && options.model.event) || 'input'
  ;(data.props || (data.props = {}))[prop] = data.model.value
  const on = data.on || (data.on = {})
  if (isDef(on[event])) {
    on[event] = [data.model.callback].concat(on[event])
  } else {
    on[event] = data.model.callback
  }
}
```

转换结果如下：

```js
data.props = {
  value: (message),
}
data.on = {
  input: function ($$v) {
    message=$$v
  }
} 
```

相当于：

```js
let vm = new Vue({
  el: '#app',
  template: '<div>' +
  '<child :value="message" @input="message=arguments[0]"></child>' +
  '<p>Message is: {{ message }}</p>' +
  '</div>',
  data() {
    return {
      message: ''
    }
  },
  components: {
    Child
  }
})
```

model可以配置

```js
let Child = {
  template: '<div>'
  + '<input :value="msg" @input="updateValue" placeholder="edit me">' +
  '</div>',
  props: ['msg'],
  model: {
    prop: 'msg',
    event: 'change'
  },
  methods: {
    updateValue(e) {
      this.$emit('change', e.target.value)
    }
  }
}
```

### slot

插槽分为：普通插槽和作用域插槽。编译是在vm.$mount的时候，先编译父组件，再编译子组件。

- 普通插槽

```js
// handle <slot/> outlets 子组件编译
function processSlotOutlet (el) {
  if (el.tag === 'slot') {
    el.slotName = getBindingAttr(el, 'name')
  }
}
//父组件编译
function processSlotContent (el) {
  let slotScope
  if (el.tag === 'template') {
    slotScope = getAndRemoveAttr(el, 'scope')
    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && slotScope) {
      warn(
        `the "scope" attribute for scoped slots have been deprecated and ` +
        `replaced by "slot-scope" since 2.5. The new "slot-scope" attribute ` +
        `can also be used on plain elements in addition to <template> to ` +
        `denote scoped slots.`,
        el.rawAttrsMap['scope'],
        true
      )
    }
    el.slotScope = slotScope || getAndRemoveAttr(el, 'slot-scope')
  } else if ((slotScope = getAndRemoveAttr(el, 'slot-scope'))) {
    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && el.attrsMap['v-for']) {
      warn(
        `Ambiguous combined usage of slot-scope and v-for on <${el.tag}> ` +
        `(v-for takes higher priority). Use a wrapper <template> for the ` +
        `scoped slot to make it clearer.`,
        el.rawAttrsMap['slot-scope'],
        true
      )
    }
    el.slotScope = slotScope
  }

  // slot="xxx"
  const slotTarget = getBindingAttr(el, 'slot')
  if (slotTarget) {
    el.slotTarget = slotTarget === '""' ? '"default"' : slotTarget
    el.slotTargetDynamic = !!(el.attrsMap[':slot'] || el.attrsMap['v-bind:slot'])
    // preserve slot as an attribute for native shadow DOM compat
    // only for non-scoped slots.
    if (el.tag !== 'template' && !el.slotScope) {
      addAttr(el, 'slot', slotTarget, getRawBindingAttr(el, 'slot'))
    }
  }

  // 2.6 v-slot syntax
  if (process.env.NEW_SLOT_SYNTAX) {
    if (el.tag === 'template') {
      // v-slot on <template>
      const slotBinding = getAndRemoveAttrByRegex(el, slotRE)
      if (slotBinding) {
        if (process.env.NODE_ENV !== 'production') {
          if (el.slotTarget || el.slotScope) {
            warn(
              `Unexpected mixed usage of different slot syntaxes.`,
              el
            )
          }
          if (el.parent && !maybeComponent(el.parent)) {
            warn(
              `<template v-slot> can only appear at the root level inside ` +
              `the receiving component`,
              el
            )
          }
        }
        const { name, dynamic } = getSlotName(slotBinding)
        el.slotTarget = name
        el.slotTargetDynamic = dynamic
        el.slotScope = slotBinding.value || emptySlotScopeToken // force it into a scoped slot for perf
      }
    } else {
      // v-slot on component, denotes default slot
      const slotBinding = getAndRemoveAttrByRegex(el, slotRE)
      if (slotBinding) {
        if (process.env.NODE_ENV !== 'production') {
          if (!maybeComponent(el)) {
            warn(
              `v-slot can only be used on components or <template>.`,
              slotBinding
            )
          }
          if (el.slotScope || el.slotTarget) {
            warn(
              `Unexpected mixed usage of different slot syntaxes.`,
              el
            )
          }
          if (el.scopedSlots) {
            warn(
              `To avoid scope ambiguity, the default slot should also use ` +
              `<template> syntax when there are other named slots.`,
              slotBinding
            )
          }
        }
        // add the component's children to its default slot
        const slots = el.scopedSlots || (el.scopedSlots = {})
        const { name, dynamic } = getSlotName(slotBinding)
        const slotContainer = slots[name] = createASTElement('template', [], el)
        slotContainer.slotTarget = name
        slotContainer.slotTargetDynamic = dynamic
        slotContainer.children = el.children.filter((c: any) => {
          if (!c.slotScope) {
            c.parent = slotContainer
            return true
          }
        })
        slotContainer.slotScope = slotBinding.value || emptySlotScopeToken
        // remove children as they are returned from scopedSlots now
        el.children = []
        // mark el non-plain so data gets generated
        el.plain = false
      }
    }
  }
}

```

子组件的init时机是在父组件执行pacth的时候，这时候父组件已经编译完成，子组件在init过程会执行initRender函数，在initRender时获取vm.$slot，拿到了 `vm.$slots` 了，回到 `renderSlot` 函数，`const slotNodes = this.$slots[name]`，我们也就能根据插槽名称获取到对应的 `vnode` 数组了，这个数组里的 `vnode` 都是在父组件创建的，这样就实现了在父组件替换子组件插槽的内容了。

父组件应用到子组件插槽里的数据都是绑定在父组件上的，它渲染成vnode的时机的上下文是父组件的实例。

当需要根据子组件的数据来决定父组件插槽的逻辑，使用作用域插槽。

- 作用域插槽

编译阶段还是父组件先编译。

```js
el.slotScope = slotScope || getAndRemoveAttr(el, 'slot-scope')
```

把slotScope属性复制在当前ast的slotScope属性上。在构造ast树的时候，执行以下逻辑：

```js
 const name = element.slotTarget || '"default"'
          ;(currentParent.scopedSlots || (currentParent.scopedSlots = {}))[name] = element
```

scopedSlots属性的ast元素节点，不会作为`children`添加到当前ast树中，而是添加到父组件的scopedSlots属性上

到`genData时`，对`scopedSlots`处理

```js
// scoped slots
  if (el.scopedSlots) {
    data += `${genScopedSlots(el, el.scopedSlots, state)},`
  }
```

相比普通插槽，作用域插槽没有children，并且执行了`_u`方法。

子组件编译时`_t`执行作用域方法`this.$scopedSlots[name]`，this.$scopedSlots参数执行是在`core/instance/render.js`中

```js
 if (_parentVnode) {
  vm.$scopedSlots = _parentVnode.data.scopedSlots || emptyObject
}
```

> 普通插槽是在父组件编译和渲染阶段生成 `vnodes`，所以数据的作用域是父组件实例，子组件渲染的时候直接拿到这些渲染好的 `vnodes`。

> 作用域插槽，父组件在编译和渲染阶段并不会直接生成 `vnodes`，而是在父节点 `vnode` 的 `data` 中保留一个 `scopedSlots` 对象，存储着不同名称的插槽以及它们对应的渲染函数，只有在编译和渲染子组件阶段才会执行这个渲染函数生成 `vnodes`，由于是在子组件环境执行的，所以对应的数据作用域是子组件实例。

### keep-alive

```js
export default {
  name: 'keep-alive',
  abstract: true,
    props: {
    include: patternTypes,
    exclude: patternTypes,
    max: [String, Number]
  },
  created () {
    this.cache = Object.create(null)
    this.keys = []
  },
  //...
     render () {
    const slot = this.$slots.default
    const vnode: VNode = getFirstComponentChild(slot)
    const componentOptions: ?VNodeComponentOptions = vnode && vnode.componentOptions
   	// ...
    return vnode || (slot && slot[0])
  }
}
```

keep-alive的本质是缓存已经创建的vnode，直接实现了render，keep-alive只处理第一个子节点，一般搭配`动态组件`或者时`router-view`。

首次渲染和普通的组件没有什么区别，再次渲染会触发`prepatch`,在`prepatch`中执行`updateChildComponent`方法，因为执行`prepatch`时，会对`slots`做重新解析，并触发组件实例的`$forceUpdate`，返回缓存中的`vnode.componentsInstance`又会执行和`patch`过程。因为`isReactivated`是true，所以不执行`created`和`mounted`等周期，会执行`reactivateComponent`,生命周期是`actived`