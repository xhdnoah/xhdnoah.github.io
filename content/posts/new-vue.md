+++
title = "Vue 初始化：从 new Vue(...) 到 DOM Tree"
date = "2019-12-18"
+++

这篇文章将从我自己的理解描述一个 Vue.js 项目是怎么从零开始初始化的主线过程，以关键函数的调用栈为线索，会略过很多繁琐的段落，分析 Vue.js 在初始化过程中的关键节点，我觉得阅读源码不能像看小说那样，线性地追寻无限的细节，而是在脑海中建立关键节点的索引和核心原理的理解，这样在需要回头具体分析某一处或者对哪部分感兴趣的时候就可以快速定位到

### 构造函数初始化

一个 Vue.js 应用由一个根实例和可嵌套的组件树构成，而组件都是可复用的 Vue 实例，可以从入口文件 `entry-runtime-with-compiler.js -> runtime/index.js -> core/index.js -> instance/index.js` 这条线找到 Vue 构造函数

```javascript
import { initMixin } from './init'
import { stateMixin } from './state'
import { renderMixin } from './render'
import { eventsMixin } from './events'
import { lifecycleMixin } from './lifecycle'
import { warn } from '../util/index'

function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}

initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)
```

Vue 把初始化、数据状态、事件、生命周期、渲染相关的逻辑拆分为单独的模块，分别挂载到 Vue.prototype 原型上，构造函数本身只执行 init 方法

除了实例方法，在 core/index.js 中的`initGlobalAPI` 还给 Vue 本身添加全局的 API，如 `Vue.set, Vue.nextTick, Vue.util` 等，其中 `Vue.options._base = Vue` 是保持对 Vue 构造函数的引用，会用作子组件的创建。

### Vue 实例初始化

```javascript
Vue.property._init = function(options?: Object){
  const vm: Component = this
  vm._uid = uid++
  vm._isVue = true
	// merge options
  if(options && options._isComponent){
    initInternalComponent(vm, options)
  } else {
    vm.$options = mergeOptions(
      resolveConstructorOptions(vm.constructor),
      options || {},
      vm
    )
  }
	// expose real self
	vm._self = vm
	initLifecycle(vm)
	initEvents(vm)
	initRender(vm)
	callHook(vm, 'beforeCreate')
	initInjections(vm)
	initState(vm)
	initProvide(vm)
	callHook(vm, 'created')
	...
	if(vm.$options.el){
		vm.$mount(vm.$options.el)
	}
}
```

**mergeOptions**

`resolveConstructorOptions` 递归获取所有组件的默认参数，`mergeOptions` 规范化用户参数然后合并用户参数和默认参数，用户参数就是你写的 name,data,props,method 和生命周期钩子等，合并后的所有参数指向 `vm.$options`

**beforeCreate**

此时还不能访问数据，可以完成和后端交互的初始化操作

**initData**

> initState 包含初始化 data props methods computed watch，这里以 initData 举例

```javascript
function initData(vm: Component){
  let data = vm.$options.data
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
  	: data || {}
  // proxy data on instance
  // check if keys are repeated
  const keys = Object.keys(data)
  const props = vm.$options.props
  const methods = vm.$options.methods
  let i = keys.length
  while (i--) {
    const key = keys[i]
    if (process.env.NODE_ENV !== 'production') {
      if (methods && hasOwn(methods, key)) {
        warn(
          `Method "${key}" has already been defined as a data property.`,
          vm
        )
      }
    }
    if (props && hasOwn(props, key)) {
      process.env.NODE_ENV !== 'production' && warn(
        `The data property "${key}" is already declared as a prop. ` +
        `Use prop default value instead.`,
        vm
      )
    } else if (!isReserved(key)) {
      proxy(vm, `_data`, key)
    }
  }
  observe(data, true)
}
```

这里从 vm.$options 中取出数据后，指向 `vm._data`，校验 data 的键值是否和 method props 重复，通过 proxy 函数代理 `vm._data` 下的私有数据，这样当我们访问 `this.msg` 就相当于访问 `this._data.msg`，最后在 data 对象中添加 Observer, 绑定数据劫持

**proxy**

```javascript
const sharedPropertyDefinition = {
	enumerable: true,
	configurable: true,
	get: noop,
	set: noop
}

export function proxy(target: Object, sourceKey: string, key: string){
	sharedPropertyDefinition.get = function proxyGetter(){
		return this[sourceKey][key]
	}
  sharedPropertyDefinition.set = function proxySetter(val){
    this[sourceKey][key] = val
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```

proxy 的实现通过 Object.defineProperty 对每一个 data 项设置 getter 和 setter

### vm 实例挂载

**vm.$mount**

```javascript
Vue.property.$mount = function(
	el?: string | Element,
	hydrating?: boolean
): Component {
	el = el && inBrowser ? query(el) : undefined
	return mountComponent(this, el, hydrating)
}
```

```javascript
export function mountComponent(
	vm: Component,
	el: ?Element,
	hydrating?: boolean
): Component {
	vm.$el = el
	...
	callHook(vm, 'beforeMount')
	let updateComponent
	updateComponent = () => {
    // 执行 _render 生成 VNode
    vm_update(vm._render(), hydrating)
	}
  ...
  // renderWatcher
  new Watcher(vm, updateComponent, noop, {
    before(){
      if(vm._isMounted){
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true)

  return vm
}
```

compiler 版本的实例挂载最终会实例化一个渲染 Watcher，在其初始化以及检测到数据变化的时候执行回调函数 `updateComponent`，`updateComponent` 调用 `vm._render()` 生成 Virtual DOM，最终调用 `vm._update` 更新到真实 DOM

**vm._render**

```javascript
Vue.prototype._render = function (): VNode {
	const vm: Component = this
	const { render, _parentVnode } = vm.$options
	if(_parentVnode){
		vm.$scopedSlots = normalizeScopedSlots(
			_parentVnode.data.scopedSlots,
			vm.$slots,
			vm.$scopedSlots
		)
	}

  vm.$vnode = _parentVnode
  let vnode
  try {
    currentRenderingInstance = vm
    // 执行 createElement
    vnode = render.call(vm._renderProxy, vm.$createElement)
  } catch(e) {
    handleError(e, vm, `render`)
    // return error render result
    ...
  } finally {
    currentRenderingInstance = null
  }
  ...
  vnode.parent = _parentVnode
  return vnode
}
```

`vm._render` 核心是调用**用户手写或编译生成的 render 函数**，并将 createElement 函数作为参数传入，当我们使用模板语法时

```vue
// 模板编译
<div id="app">
 {{ message }}
</div>
```

template 最终都会编译成 render 函数，相当于

```javascript
// 手写 render 函数, createElement 作为参数传入创建 VNode
var app = new Vue({
	el: '#app',
	render(createElement){
		return createElement('div', {
			attrs: {
				id: '#app'
			}
		}, this.message)
	}
})
```

**renderProxy**

render 调用绑定了 `vm._renderProxy` 中的 this 指向，`vm._renderProxy` 在生产环境指向 vm，在开发环境对 vm 的读写操作进行劫持并检测错误

```javascript
const hasHandler = {
  ...
  warnNonPresent(target, key)
}

const getHandler = {
  ...
  warnNonPresent(target, key)
}

initProxy = function initProxy (vm) {
	if(hasProxy){
		const options = vm.$options
    const handlers = options.render && options.render._withStripped
    	? getHandler
    	: hasHandler
    vm._renderProxy = new Proxy(vm, handlers)
	} else {
    vm._renderProxy = vm
  }
}
```

`warnNonPresent()` 表示属性或方法未在实例上定义

**Virtual DOM**

Virtual DOM 是一个用 JS 对象描述的虚拟 DOM节点，Vue.js 用 VNode 类来描述 Virtual DOM

```javascript
export default class VNode {
  tag: string | void;
  data: VNodeData | void;
  children: ?Array<VNode>;
  text: string | void;
  elm: Node | void;
  ns: string | void;
  context: Component | void; // rendered in this component's scope
  key: string | number | void;
  componentOptions: VNodeComponentOptions | void;
  componentInstance: Component | void; // component instance
  parent: VNode | void; // component placeholder node

  // strictly internal
  raw: boolean; // contains raw HTML? (server only)
  isStatic: boolean; // hoisted static node
  isRootInsert: boolean; // necessary for enter transition check
  isComment: boolean; // empty comment placeholder?
  isCloned: boolean; // is a cloned node?
  isOnce: boolean; // is a v-once node?
  asyncFactory: Function | void; // async component factory function
  asyncMeta: Object | void;
  isAsyncPlaceholder: boolean;
  ssrContext: Object | void;
  fnContext: Component | void; // real context vm for functional nodes
  fnOptions: ?ComponentOptions; // for SSR caching
  fnScopeId: ?string; // functional scope id support

  constructor (
    tag?: string,
    data?: VNodeData,
    children?: ?Array<VNode>,
    text?: string,
    elm?: Node,
    context?: Component,
    componentOptions?: VNodeComponentOptions,
    asyncFactory?: Function
  ) {
    this.tag = tag
    this.data = data
    this.children = children
    this.text = text
    this.elm = elm
    this.ns = undefined
    this.context = context
    this.fnContext = undefined
    this.fnOptions = undefined
    this.fnScopeId = undefined
    this.key = data && data.key
    this.componentOptions = componentOptions
    this.componentInstance = undefined
    this.parent = undefined
    this.raw = false
    this.isStatic = false
    this.isRootInsert = true
    this.isComment = false
    this.isCloned = false
    this.isOnce = false
    this.asyncFactory = asyncFactory
    this.asyncMeta = undefined
    this.isAsyncPlaceholder = false
  }

  // DEPRECATED: alias for componentInstance for backwards compat.
  /* istanbul ignore next */
  get child (): Component | void {
    return this.componentInstance
  }
}
```

Vue.js 中的 VNode 还分为**占位符 VNode** 和 **渲染 VNode**，占位符 VNode 就是组件在渲染过程中用来占位的父 VNode，可以理解为组件的根节点，渲染 VNode 就是有真实 DOM 映射关系的 DOM 节点。在实例属性中， `vm.$vnode` 指向占位符节点，`vm._vnode` 指向渲染节点

#### 创建 VNode

Vue.js 利用 createElement 方法创建 VNode，createElement 封装了 _createElement 方法，后者主要完成两件事：子节点的规范化和 VNode 的实例化，以 vuejs.org 官网的 demo 为例

```javascript
// @returns {VNode}
createElement(
  // {String | Object | Function}
  // An HTML tag name, component options, or async
  // function resolving to one of these. Required.
  'div',

  // {Object}
  // A data object corresponding to the attributes
  // you would use in a template. Optional.
  {
    // (see details in the next section below)
  },

  // {String | Array}
  // Children VNodes, built using `createElement()`,
  // or using strings to get 'text VNodes'. Optional.
  [
    'Some text comes first.',
    createElement('h1', 'A headline'),
    createElement(MyComponent, {
      props: {
        someProp: 'foobar'
      }
    })
  ]
)
```

```javascript
export function _createElement (
  context: Component,
  tag?: string | Class<Component> | Function | Object,
  data?: VNodeData,
  children?: any,
  normalizationType?: number
): VNode | Array<VNode> {
  if (isDef(data) && isDef((data: any).__ob__)) {
    process.env.NODE_ENV !== 'production' && warn(
      `Avoid using observed data object as vnode data: ${JSON.stringify(data)}\n` +
      'Always create fresh vnode data objects in each render!',
      context
    )
    return createEmptyVNode()
  }
  // object syntax in v-bind
  if (isDef(data) && isDef(data.is)) {
    tag = data.is
  }
  if (!tag) {
    // in case of component :is set to falsy value
    return createEmptyVNode()
  }
  // warn against non-primitive key
  if (process.env.NODE_ENV !== 'production' &&
    isDef(data) && isDef(data.key) && !isPrimitive(data.key)
  ) {
    if (!__WEEX__ || !('@binding' in data.key)) {
      warn(
        'Avoid using non-primitive value as key, ' +
        'use string/number value instead.',
        context
      )
    }
  }
  // support single function children as default scoped slot
  if (Array.isArray(children) &&
    typeof children[0] === 'function'
  ) {
    data = data || {}
    data.scopedSlots = { default: children[0] }
    children.length = 0
  }
  if (normalizationType === ALWAYS_NORMALIZE) {
    children = normalizeChildren(children)
  } else if (normalizationType === SIMPLE_NORMALIZE) {
    children = simpleNormalizeChildren(children)
  }
  let vnode, ns
  if (typeof tag === 'string') {
    let Ctor
    ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
    if (config.isReservedTag(tag)) {
      // platform built-in elements
      vnode = new VNode(
        config.parsePlatformTagName(tag), data, children,
        undefined, undefined, context
      )
    } else if (isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
      // component
      vnode = createComponent(Ctor, data, context, children, tag)
    } else {
      // unknown or unlisted namespaced elements
      // check at runtime because it may get assigned a namespace when its
      // parent normalizes children
      vnode = new VNode(
        tag, data, children,
        undefined, undefined, context
      )
    }
  } else {
    // direct component options / constructor
    vnode = createComponent(tag, data, context, children)
  }
  if (Array.isArray(vnode)) {
    return vnode
  } else if (isDef(vnode)) {
    if (isDef(ns)) applyNS(vnode, ns)
    if (isDef(data)) registerDeepBindings(data)
    return vnode
  } else {
    return createEmptyVNode()
  }
}
```

**normalizeChildren**

```javascript
if(normalizationType === ALWAYS_NORMALIZE){
    // 将嵌套的 children 数组拍平
    children = normalizationChildren(children)
} else if (normalizationType === SIMPLE_NORMALIZE){
    // simpleNormalize 将二维数组拍平
    children = simpleNormalizeChildren(children)
}
```

最终都是将 children 变成类型为 VNode 的一维数组

**vnode**

```javascript
let vnode, ns
if (typeof tag === 'string') {
  let Ctor
  ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
  if (config.isReservedTag(tag)) {
    // platform built-in elements
    vnode = new VNode(
      config.parsePlatformTagName(tag), data, children,
      undefined, undefined, context
    )
  } else if (isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
    // component
    vnode = createComponent(Ctor, data, context, children, tag)
  } else {
    // unknown or unlisted namespaced elements
    // check at runtime because it may get assigned a namespace when its
    // parent normalizes children
    vnode = new VNode(
      tag, data, children,
      undefined, undefined, context
    )
  }
} else {
  // direct component options / constructor
  vnode = createComponent(tag, data, context, children)
}
```

创建 VNode 时对 tag 进行判断，如果 tag 是 string 类型还是 Component 类型，string 类型下再判断是内置节点还是已注册的组件名。根据 tag 类型决定创建普通 VNode 还是组件 VNode，组件初始化的过程在

**update**

回到 updateComponent，创建好的 VNode 还需要通过 `vm._update` 渲染为真实 DOM。`vm._update` 的核心是调用 `vm.__patch__` ，在首次渲染和数据更新的时候执行

```javascript
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
  const vm: Component = this
  const prevEl = vm.$el
  const prevVnode = vm._vnode
  const prevActiveInstance = activeInstance
  activeInstance = vm
  vm._vnode = vnode
  // Vue.prototype.__patch__ is injected in entry points
  // based on the rendering backend used.
  if (!prevVnode) {
    // initial render
    vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
  } else {
    // updates
    vm.$el = vm.__patch__(prevVnode, vnode)
  }
  activeInstance = prevActiveInstance
  // update __vue__ reference
  if (prevEl) {
    prevEl.__vue__ = null
  }
  if (vm.$el) {
    vm.$el.__vue__ = vm
  }
  // if parent is an HOC, update its $el as well
  if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
    vm.$parent.$el = vm.$el
  }
  // updated hook is called by the scheduler to ensure that children are
  // updated in a parent's updated hook.
}
```

**patch 与函数柯里化**

> 《Mostly adequate guide》总结函数柯里化——— 只传递给函数一部分参数来调用它，让它返回一个函数去处理剩下的参数；柯里化为实现多参函数提供了一个递归降解的实现思路——把接受多个参数的函数变换成接受一个单一参数（最初函数的第一个参数）的函数，并且返回接受余下的参数而且返回结果的新函数

Vue.js 可以跨平台运行，不同的平台拥有不同的 nodeOps 方法和modules 属性 。利用函数柯里化可以避免每次执行 patch 都要`if...else...`判断平台条件，通过 `createPatchFunction` 在返回真正的 patch 函数之前就将平台之间的差异一次性抹平

```javascript
// nodeOps: 操作 DOM 方法
// modules: DOM 各类属性、类、事件、样式、钩子函数等
export const patch: Function = createPatchFunction({ nodeOps, modules})

//patch.js
export function createPatchFunction (backend){
  let i, j
  const cbs = {}

  const {modules, nodeOps } = backend

  for(i = 0; i < hooks.length; ++i){
    cbs[hooks[i]] = []
    for(j = 0; j < modules.length; ++j){
      if(isDef(modules[j][hooks[i]])){
        cbs[hooks[i]].push(modules[j][hooks[i]])
      }
    }
  }
  ... 辅助函数
  // oldVnode: 不存在/一个真实 DOM
  // vnode: _render 执行返回的 VNode 节点
  return function patch(oldVnode, vnode, hyrating, removeOnly){
    const isRealElement = isDef(oldVnode.nodeType)
    if(!isRealElement && sameVnode(oldVnode, vnode)){
      patchVnode(oldVnode, vnode, insertedVnodeQueue, removeOnly)
    } else {
      if(isRealElement){
        if(oldVnode.nodeType === 1 && oldVnode.hasAttribute(SSR_ATTR)){
          oldVnode.removeAttribute(SSR_ATTR)
          hydrating = true
        }
        if(isTrue(hydrating)){
          //...SSR
        }
        // create an empty node and replace it
        oldVnode = emptyNodeAt(oldVnode)
      }

      // replacing existing element
      const oldElm = oldVnode.elm
      const parentElm = nodeOps.parentNode(oldElm)

      createElm(
      	vnode,
        insertedVnodeQueue,
        ...,
        nodeOps.nextSibling(oldElm)
      )
    }
  }
}
```

在首次初始化的时候，`oldVnode` 即 `vm.$el` 实际上是一个容器，`isRealElement` 为 true，再通过 `empthNodeAt` 将 `oldVnode` 转换为 `VNode` ，最后调用 `createElm ` 创建真实 DOM 插入到父节点中

**createElm**

```javascript
function createElm (
	vnode,
	insertedVnodeQueue,
	parentElm,
	refElm,
	nested,
	ownerArray,
	index
){
	if(isDef(vnode.elm) && isDef(ownerArray)){
		vnode = ownerArray[index] = cloneVNode(vnode)
	}
	if(createComponent(vnode, insertedVnodeQueue, parentElm, refElm)){
		return
	}

	const data = vnode.data
	const children = vnode.children
	const tag = vnode.tag
	if(isDef(tag)){
		...
	}
	// 创建占位符元素
	vnode.elm = nodeOps.createElement(tag, vnode)
	setScope(vnode)

  createChildren(vnode, children, insertedVnodeQueue)
  if(isDef(data)){
    invokeCreateHooks(vnode, insertedVnodeQueue)
  }
  insert(parentElm, vnode.elm, refElm)
}
```

**createChildren**

`createChildren` 创建子元素，深度优先遍历子虚拟节点，递归调用 `createElm`，遍历过程中把 `vnode.elm` 作为父容器的 DOM 节点占位符传入

```javascript
function createChildren(vnode, children, insertedVnodeQueue){
	if(Array.isArray(children)){
		if(process.env.NODE_ENV !== 'production'){
			checkDuplicateKeys(children)
		}
		for(let i = 0; i < children.length; ++i){
			createElm(children[i], insertedVnodeQueue, vnode.elm, null, true, children, i)
		}
	} else if(isPrimitive(vnode.text)){
    nodeOps.appendChild(vnode.elm, nodeOps.createTextNode(String(vnode.text)))
  }
}
```

**invokeCreateHooks**

执行所有 create 钩子并把 `vnode` push 到 `insertedVnodeQueue`

```
function invokeCreateHooks(vnode, insertedVnodeQueue){
	for(let i = 0; i < cbs.create.length; ++i){
		cbs.create[i](emptyNode, vnode)
	}
  i = vnode.data.hook	// Reuse variable
  if(isDef(i)){
    if(isDef(i.create)) i.create(emptyNode, vnode)
    if(isDef(i.insert)) insertedVnodeQueue.push(vnode)
  }
}
```

**insert**

最终调用 `insert` 方法把 DOM 插入父节点，DOM 节点的插入顺序是先子后父，直到整个 DOM 树插入 `oldVnode.elm` 的父元素 Body

```javascript
function insert(parent, elm, ref){
	if(isDef(parent)){
		if(isDef(ref)){
			if(ref.parentNode === parent){
				nodeOps.insertedBefore(parent, elm, ref)
			}
		} else {
			nodeOps.appendChild(parent, elm)
		}
	}
}
```

初始化过程到了这里终于渲染成了最终的真实 DOM，关键节点的流程图如下所示，其中从 `createElement` 开始对组件的初始化过程在下篇介绍

```
    +----------+ init +--------+    +---------+    +-----------------+
    | new Vue()+------> $mount +----> compile +----> render function +--+
    +----------+      +--------+    +---------+    +-----------------+  |
                                                                        |
                                                                        |
+-----+    +-----------+     +-------+ update +-------+    +------------v--+
| DOM <----+ createElm <-----+ patch <--------+ vnode <----+ createElement |
+-----+    +-----------+     +-------+        +-------+    +---------------+

```
