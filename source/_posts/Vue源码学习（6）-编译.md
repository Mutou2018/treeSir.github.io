---
title: Vue源码学习（6）-编译
date: 2021-5-13 19:49:41
tags: vue
cover: https://raw.githubusercontent.com/DaMu2018/cloudimg/main/data/vue.png
top_img: https://raw.githubusercontent.com/DaMu2018/cloudimg/main/data/ready-to-go.jpg
---

### 编译

入口是`src/platforms/web/entry-runtime-with-compiler.js`。编译的入口就是

```js
const { render, staticRenderFns } = compileToFunctions(template, {
        outputSourceRange: process.env.NODE_ENV !== 'production',
        shouldDecodeNewlines,
        shouldDecodeNewlinesForHref,
        delimiters: options.delimiters,
        comments: options.comments
}, this)
      options.render = render
      options.staticRenderFns = staticRenderFns
```

把template转译成为render函数

```js
export const createCompiler = createCompilerCreator(function baseCompile (
  template: string,
  options: CompilerOptions
): CompiledResult {
  const ast = parse(template.trim(), options)
  if (options.optimize !== false) {
    optimize(ast, options)
  }
  const code = generate(ast, options)
  return {
    ast,
    render: code.render,
    staticRenderFns: code.staticRenderFns
  }
})
```

parse将html转换为ast（abstract syntax tree），抽象语法树。基于`baseCompile`,vue会将template进行不同的处理。

- parse解析

  解析流程为：

  从options获取方法和配置 --> parseHTML解析html模板 --> 解析模板分为解析开始标签，闭合标签，文本等，vue的关键字等。

  AST元素节点有3种类型：

  `type 1`:普通元素，

  `type 2`:表达式

  `type 3`:纯文本

  生成树状结构的ast，拥有自己的属性。

- optimize 优化

  将static类型的锁定，不再改变dom，标记静态节点和标记静态根` markStatic()`和`markStaticRoots`,

  `v-once`和`static`都是静态不再改变dom的。当type==3时，表示是纯文本的。`v-pre`也是静态的。

- 把优化后的ast树，转换成可执行的代码

  ```js
  const code = generate(ast, options)
  ```

  将`code`转换成render代码串。

> `render` 代码串通过 `new Function` 的方式转换成可执行的函数，赋值给 `vm.options.render`，这样当组件通过 `vm._render` 的时候，就会执行这个 `render` 函数。

generate的定义：

```js
export function generate (
  ast: ASTElement | void,
  options: CompilerOptions
): CodegenResult {
  const state = new CodegenState(options)
  // fix #11483, Root level <script> tags should not be rendered.
  const code = ast ? (ast.tag === 'script' ? 'null' : genElement(ast, state)) : '_c("div")'
  return {
    render: `with(this){return ${code}}`,
    staticRenderFns: state.staticRenderFns
  }
}
```

执行`genElement()`函数。

```js
export function genElement (el: ASTElement, state: CodegenState): string {
  if (el.parent) {
    el.pre = el.pre || el.parent.pre
  }

  if (el.staticRoot && !el.staticProcessed) {
    return genStatic(el, state)
  } else if (el.once && !el.onceProcessed) {
    return genOnce(el, state)
  } else if (el.for && !el.forProcessed) {
    return genFor(el, state)
  } else if (el.if && !el.ifProcessed) {
    return genIf(el, state)
  } else if (el.tag === 'template' && !el.slotTarget && !state.pre) {
    return genChildren(el, state) || 'void 0'
  } else if (el.tag === 'slot') {
    return genSlot(el, state)
  } else {
    // component or element
    let code
    if (el.component) {
      code = genComponent(el.component, el, state)
    } else {
      let data
      if (!el.plain || (el.pre && state.maybeComponent(el))) {
        data = genData(el, state)
      }

      const children = el.inlineTemplate ? null : genChildren(el, state, true)
      code = `_c('${el.tag}'${
        data ? `,${data}` : '' // data
      }${
        children ? `,${children}` : '' // children
      })`
    }
    // module transforms
    for (let i = 0; i < state.transforms.length; i++) {
      code = state.transforms[i](el, code)
    }
    return code
  }
}
```

区分了各种情况，包括：Static、Once、For、If、template、slot、component、data情况。

`genDate`：

```js
export function genData (el: ASTElement, state: CodegenState): string {
  let data = '{'

  // directives first.
  // directives may mutate the el's other properties before they are generated.
  const dirs = genDirectives(el, state)
  if (dirs) data += dirs + ','

  // key
  if (el.key) {
    data += `key:${el.key},`
  }
  // ref
  if (el.ref) {
    data += `ref:${el.ref},`
  }
  if (el.refInFor) {
    data += `refInFor:true,`
  }
  // pre
  if (el.pre) {
    data += `pre:true,`
  }
  // record original tag name for components using "is" attribute
  if (el.component) {
    data += `tag:"${el.tag}",`
  }
  // module data generation functions
  for (let i = 0; i < state.dataGenFns.length; i++) {
    data += state.dataGenFns[i](el)
  }
  // attributes
  if (el.attrs) {
    data += `attrs:${genProps(el.attrs)},`
  }
  // DOM props
  if (el.props) {
    data += `domProps:${genProps(el.props)},`
  }
  // event handlers
  if (el.events) {
    data += `${genHandlers(el.events, false)},`
  }
  if (el.nativeEvents) {
    data += `${genHandlers(el.nativeEvents, true)},`
  }
  // slot target
  // only for non-scoped slots
  if (el.slotTarget && !el.slotScope) {
    data += `slot:${el.slotTarget},`
  }
  // scoped slots
  if (el.scopedSlots) {
    data += `${genScopedSlots(el, el.scopedSlots, state)},`
  }
  // component v-model
  if (el.model) {
    data += `model:{value:${
      el.model.value
    },callback:${
      el.model.callback
    },expression:${
      el.model.expression
    }},`
  }
  // inline-template
  if (el.inlineTemplate) {
    const inlineTemplate = genInlineTemplate(el, state)
    if (inlineTemplate) {
      data += `${inlineTemplate},`
    }
  }
  data = data.replace(/,$/, '') + '}'
  // v-bind dynamic argument wrap
  // v-bind with dynamic arguments must be applied using the same v-bind object
  // merge helper so that class/style/mustUseProp attrs are handled correctly.
  if (el.dynamicAttrs) {
    data = `_b(${data},"${el.tag}",${genProps(el.dynamicAttrs)})`
  }
  // v-bind data wrap
  if (el.wrapData) {
    data = el.wrapData(data)
  }
  // v-on data wrap
  if (el.wrapListeners) {
    data = el.wrapListeners(data)
  }
  return data
}

```

genData首先生成`const dirs = genDirectives(el, state)`指令可能会引起其他属性的改变，所以顺序上会先生成。

### event

```vue
<div @click='handleClick()'>click me<div/>
methods:{
	handleClick(){
		console.log('click')
	}
}
```

编译：src/compiler/parser/index.js中。对标签属性的处理过程中，如果是指令会通过`parseModifier`解析出修饰符，如果是事件的指令，则执行`addhandler（）`,在`helper.js`中根据不同的修饰符来进行处理。

在`codegen`(把优化后的ast转换为可执行的代码)，会在`genData`生成data数据，genData调用genHandlers数据。modifyCode:

```js
const modifierCode: { [key: string]: string } = {
  stop: '$event.stopPropagation();',
  prevent: '$event.preventDefault();',
  self: genGuard(`$event.target !== $event.currentTarget`),
  ctrl: genGuard(`!$event.ctrlKey`),
  shift: genGuard(`!$event.shiftKey`),
  alt: genGuard(`!$event.altKey`),
  meta: genGuard(`!$event.metaKey`),
  left: genGuard(`'button' in $event && $event.button !== 0`),
  middle: genGuard(`'button' in $event && $event.button !== 1`),
  right: genGuard(`'button' in $event && $event.button !== 2`)
```

### DOM事件

- 原生dom事件

  `patch`的时候执行各种`modules`的钩子函数。在`patch`过程中的创建和更新阶段都会执行`updateDOMListeners`，添加事件的方法

  ```js
  function add (
    name: string,
    handler: Function,
    capture: boolean,
    passive: boolean
  ) {
    // firefox的微任务处理bug
    // ...
    target.addEventListener(
      name,
      handler,
      supportsPassive
        ? { capture, passive }
        : capture
    )
  }
  ```

- 自定义事件

  在`src/core/instance/events.js`

  ```js
  export function eventsMixin (Vue: Class<Component>) {
    const hookRE = /^hook:/
    Vue.prototype.$on = function (event: string | Array<string>, fn: Function): Component {
      const vm: Component = this
      if (Array.isArray(event)) {
        for (let i = 0, l = event.length; i < l; i++) {
          vm.$on(event[i], fn)
        }
      } else {
        (vm._events[event] || (vm._events[event] = [])).push(fn)
        // optimize hook:event cost by using a boolean flag marked at registration
        // instead of a hash lookup
        if (hookRE.test(event)) {
          vm._hasHookEvent = true
        }
      }
      return vm
    }
  
    Vue.prototype.$once = function (event: string, fn: Function): Component {
      const vm: Component = this
      function on () {
        vm.$off(event, on)
        fn.apply(vm, arguments)
      }
      on.fn = fn
      vm.$on(event, on)
      return vm
    }
  
    Vue.prototype.$off = function (event?: string | Array<string>, fn?: Function): Component {
      const vm: Component = this
      // all
      if (!arguments.length) {
        vm._events = Object.create(null)
        return vm
      }
      // array of events
      if (Array.isArray(event)) {
        for (let i = 0, l = event.length; i < l; i++) {
          vm.$off(event[i], fn)
        }
        return vm
      }
      // specific event
      const cbs = vm._events[event]
      if (!cbs) {
        return vm
      }
      if (!fn) {
        vm._events[event] = null
        return vm
      }
      // specific handler
      let cb
      let i = cbs.length
      while (i--) {
        cb = cbs[i]
        if (cb === fn || cb.fn === fn) {
          cbs.splice(i, 1)
          break
        }
      }
      return vm
    }
  
    Vue.prototype.$emit = function (event: string): Component {
      const vm: Component = this
      if (process.env.NODE_ENV !== 'production') {
        const lowerCaseEvent = event.toLowerCase()
        if (lowerCaseEvent !== event && vm._events[lowerCaseEvent]) {
          tip(
            `Event "${lowerCaseEvent}" is emitted in component ` +
            `${formatComponentName(vm)} but the handler is registered for "${event}". ` +
            `Note that HTML attributes are case-insensitive and you cannot use ` +
            `v-on to listen to camelCase events when using in-DOM templates. ` +
            `You should probably use "${hyphenate(event)}" instead of "${event}".`
          )
        }
      }
      let cbs = vm._events[event]
      if (cbs) {
        cbs = cbs.length > 1 ? toArray(cbs) : cbs
        const args = toArray(arguments, 1)
        const info = `event handler for "${event}"`
        for (let i = 0, l = cbs.length; i < l; i++) {
          invokeWithErrorHandling(cbs[i], vm, args, vm, info)
        }
      }
      return vm
    }
  }
  ```

  把所有事件存储在vm._events中，当执行到$emit时，找到执行函数然后遍历执行。

  > `vm.$emit` 是给当前的 `vm` 上派发的实例，之所以我们常用它做父子组件通讯，_**是因为它的回调函数的定义是在父组件中**_，对于我们这个例子而言，当子组件的 `button` 被点击了，它通过 `this.$emit('select')` 派发事件，那么子组件的实例就监听到了这个 `select` 事件，并执行它的回调函数——定义在父组件中的 `selectHandler` 方法，这样就相当于完成了一次父子组件的通讯。

原生DOM事件和自定义事件区别在于add和remove的不同，自定义事件的派发是往当前实例进行派发，但是利用父组件环境定义回调函数实现父子通信。