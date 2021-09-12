+++
title = "Vue 初始化：从 createElement 到 Component Tree"
date = "2019-12-19"
+++

本文介绍 Vue 组件的初始化过程

**createComponent**

在 `_createElement` 方法中，程序对参数 `tag` 进行判断，如果是一个原生的标签，就会按上文的分析生成一个普通的 VNode 节点；如果是一个已注册的组件名，就会走到 `createComponent` 创建一个组件 VNode

```javascript
export function createComponent(
	Ctor: Class<Component> | Function | Object | void,
	data: ?VNodeData,
	context: Component,
	children: ?Array<VNode>,
	tag?: string
): VNode | Array<VNode> | void{
  // baseCtor 指向 Vue
  const baseCtor = context.$options._base
  if(isObject(Ctor)){
    Ctor = baseCtor.extend(Ctor)
  }

	// async component
	let asyncFactory
  if(isUndef(Ctor.cid)){
    asyncFactor = Ctor
    Ctor = resolveAsyncComponent(asyncFactory, baseCtor, context)
    if(Ctor === undefined){
      return createAsyncPlaceholder(...)
    }
  }

  data = data || {}

  resolveConstructorOptions(Ctor)

  if(isDef(data.model)){
    transformModel(Ctor.options, data)
  }

  installComponentHooks(data)
  // return a placeholder vnode
  const name = Ctor.options.name || tag
  const vnode = new VNode(
    `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
    data, undefined, undefined, undefined, context,
    { Ctor, propsData, listeners, tag, children },
    asyncFactory
  )

  return vnode
}
```

**Vue.extend**

一个 Vue 组件就是对 Vue 对象的继承和扩展，Vue 使用 `Vue.extend` 构造一个 `Vue` 子类，使用原型继承方式生成一个子类的构造器 Sub 并返回，对 Sub 对象本身扩展属性，并初始化 props computed 等选项，最后将构造器缓存以便于复用

```javascript
Vue.extend = function(extendOptions: Object): Function {
	extendOptions = extendOptios || {}
	const Super = this
	const SuperId = Super.cid
	const cachedCtors = extendOptions._Ctor || (extendOptions._Ctor = {})
	if(cachedCtors[SuperId]){
		return cachedCtors[SuperId]
	}

	const Sub = function VueComponent(options){
		this._init(options)
	}
	Sub.prototype = Object.create(Super.prototype)
	Sub.prototype.constructor = Sub
	Sub.cid = cid++
	Sub.options = mergeOptions(
		Super.options,
		extendOptions
	)
	Sub['super'] = Super
	if(Sub.options.props){
		initProps(Sub)
	}
	if(Sub.options.computed){
		initComputed(Sub)
	}
	Sub.extend = Super.extend
	Sub.mixin = Super.mixin
	Sub.use = Sub.use

	cachedCtors[SuperId] = Sub
	return Sub
}
```

当实例化 Sub 时，就会执行 `this._init ` 再次走到 Vue 实例化的初始化逻辑

**实例化组件 VNode**

```javascript
const name = Ctor.options.name || tag
const vnode = new VNode(
  `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
  data, undefined, undefined, undefined, context,
  { Ctor, propsData, listeners, tag, children },
  asyncFactory
)
return vnode
```

返回的组件 VNode 是一个没有 children 的占位符 `vnode`

**createComponent**

和 `createElement` 中的同名方法不一样，这里的 `createComponent` 是执行到 `createElm` 中用来转换真实 DOM 节点的方法

```javascript
function createElm(
	vnode,
	insertedVnodeQueue,
	parentElm,
	refElm,
	nested,
	ownerArray,
	index
) {
	//...
	if(createComponent(vnode, insertedVnodeQueue, parentElm, refElm)){
		return
	}
  //...
}
```

`createComponent` 方法通过 `vnode.data`  校验传入的是否为组件 VNode，读取 data 中的 init 钩子函数并执行

```javascript
function createComponent(vnode, insertedVnodeQueue, parentElm, refElm){
	let i = vnode.data
	if(isDef(i)){
		const isReactivated = isDef(vnode.componentInstance) && i.keepAlive
    // i.hook 且 hook.init 存在则判断为组件 VNode
		if(isDef(i = i.hook) && isDef(i = i.init)){
      // 执行 init
			i(vnode, false)
		}
	}
}
```

**init hook**

```javascript
const componentVNodeHooks = {
	init(vnode: VNodeWithData, hydrating: boolean): ?boolean{
		if(
			vnode.componentInstance &&
			!vnode.componentInstance._isDestroyed &&
			vnode.data.keepAlive
		){
			// ...keepAlive
		} else {
			const child = vnode.componentInstance = createComponentInstanceForVnode(
        // 父的占位符 VNode
				vnode,
        // vm 实例
				activeInstance
			)
			child.$mount(hydrating ? vnode.elm : undefined, hydrating)
		}
	}
}
```

**createComponentInstanceForVnode**

Init 钩子通过 `createComponentInstanceForVnode` 创建一个 Vue 实例，调用 `$mount` 挂载子组件，选项中的 `_parentVnode` 表示组件的占位节点，`parent` 表示当前激活组件的实例，Vue.js 通过一个全局变量 `activeInstance` 在上下文中保存当前激活的实例

 ```javascript
function createComponentInstanceForVnode (
	vnode: any,
	parent: any
): Component {
	const options: InternalComponentOptions = {
		_isComponent: true,
		_parentVnode: vnode,	// 占位节点
		parent // 当前激活组件的实例（子组件的父 vm 实例）
	}
  // check inline-template render functions
    ...
	return new vnode.componentOptions.Ctor(options)
}
 ```

这里的 `vnode.componentOptions.Ctor` 对应的就是子组件的构造函数，子组件的实例化开始，执行到 `_init` ，组件 VNode 和普通 VNode 初始化的不同之处在合并 options 和挂载过程

```javascript
Vue.prototype._init = function(options?: Object){
	const vm: Component = this
	if(options && options._isComponent){
		initInternalComponent(vm, options)
	} else {
		vm.$options = mergeOptions(
			resolveConstructorOptions(vm.constructor),
			options || {},
			vm
		)
	}
	//...
	if(vm.$options.el){
		vm.$mount(vm.$options.el)
	}
}
```

**initInternalComponent**

```javascript
export function initInternalComponent (vm: Component, options: InternalComponentOptions){
	const opts = vm.$options = Object.create(vm.constructor.options)
  const parentVnode = options._parentVnode
  opts.parent = options.parent
  opts._parentVnode = parentVnode

  const vnodeComponentOptions = parentVnode.componentOptions
  opts.propsData = vnodeComponentOptions.propsData
  opts._parentListeners = vnodeComponentOptions.listeners
  opts._renderChildren = vnodeComponentOptions.children
  opts._componentTag = vnodeComponentOptions.tag

  if (options.render) {
    opts.render = options.render
    opts.staticRenderFns = options.staticRenderFns
  }
}
```

其中 `opts.parent = options.parent opts._parentVnode = parentVnode` 是把之前通过 `createComponentInstanceForVnode` 传入的参数合并到内部的选项 $options 中

**child.$mount**

在 `_init` 函数最后有

```js
if (vm.$options.el) {
   vm.$mount(vm.$options.el)
}
```

组件初始化不传 el，自己执行了 $mount， `compoenentVNodeHooks` 的 `init` 钩子函数在实例化 `_init` 后，接着执行 `child.$mount(hydrating ? vnode.elm : undefined, hydrating)`，最终调用 `mountComponent`，进而执行 `vm._render()`

**vm._render()**

```javascript
Vue.prototype._render = function(): VNode{
	const vm: Component = this
	const { render, _parentVnode } = vm.$options

	vm.$vnode = _parentVnode
	let vnode
	try{
		vnode = render.call(vm._renderProxy, vm.$createElement)
	} catch(e){
		...
	}

	vnode.parent = _parentVnode
	return vnode
}
```

这里的 `_parentVnode` 就是当前组件父的占位符 VNode， `render.call()` 生成的 `vnode` 是当前组件的渲染 VNode，`vnode` 的 `parent` 指向了 `_parentVnode`，也就是 `vm.$vnode`

执行完 `vm._render` 生成渲染 VNode 后，接着执行 `vm._update`

**vm._update()**

```javascript
export let activeInstance: any = null
// vnode 渲染 VNode
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean){
	const vm: Component = this
	const prevEl = vm.$el
	const prevVnode = vm._vnode
	const prevActiveInstance = activeInstance
	activeInstance = vm
	vm._vnode = vnode
	if(!prevVnode){
		// initial render
		vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false)
	} else {
    // updates
    vm.$el = vm.__patch__(prevVnode, vnode)
  }
  activeInstance = prevActiveInstance
  // update __vue__ reference
  if(prevEl){
    prevEl.__vue__ = null
  }
  if(vm.$el){
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

Vue 2.6 对 `lifecycleMixin()` 进一步抽象，把 reset 操作作为函数的返回值保存在 lifecycleMixin 作用域中

```javascript
function setActiveInstance(vm: Component){
	const prevActiveInstance = activeInstance
	activeInstance = vm
	return () => {
		activeInstance = prevActiveInstance
	}
}

function lifecycleMixin(Vue: Class){
  Vue.prototype._update = function(vnode, hydrating){
    //...
    const restoreActiveInstance = setActiveInstance(vm)
    vm._vnode = vnode
    if (!prevVnode) {
      // initial render
      vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
    } else {
      // updates
      vm.$el = vm.__patch__(prevVnode, vnode)
    }
    restoreActiveInstance()
  }
}
```

`vm._vnode = vnode` 其中 `vnode` 是通过`vm._render()` 返回的组件渲染VNode，`vm._vnode` 和 `vm.$vnode` 的关系就是一种父子关系，`vm._vnode.parent = vm.$vnode`

`activeInstance` 作用是保持当前上下文正在激活的 Vue 实例，是在 `lifecycle` 模块的全局变量，在之前我们调用 `createComponentInstanceForVnode` 方法的时候从 `lifecycle` 模块获取，并作为参数传入。Vue 整个初始化是一个深度遍历的过程，在实例化子组件的过程中，它需要知道上下文的 Vue 实例是什么，并把它作为子组件的父 Vue 实例。在对子组件的实例化过程会调用 `initInternalComponentvm, options)` 合并 `options`，把 `parent` 存到 `vm.$options` 中，在 $mount 前会调用 `initLifecycle(vm)` 建立父子关系

**initLifecycle**

```javascript
export function initLifecycle(vm: Component){
	const options = vm.$options
	let parent = options.parent
	if(parent && !options.abstract){
		while(parent.$options.abstract && parent.$parent){
      parent = parent.$parent
    }
    parent.$children.push(vm)
	}
  vm.$parent = parent
  //...
}
```

在 `vm._update` 的过程中，把当前的 `vm` 赋值给 `activeInstance`，同时通过  `prevActiveInstance` 保存上一次的 `activeInstance`。实际上`prevActiveInstance` 和当前的 `vm` 是一个父子关系，当一个 `vm` 实例完成它的所有子树的 patch/update 过程后，`activeInstance` 会回到它的父实例，这样就完美地保证了 `createComponentInstanceForVnode` 整个深度遍历过程中，我们在实例化子组件的时候能传入当前子组件的父 Vue 实例，并在 `_init` 的过程中，通过 `vm.$parent` 把这个父子关系保存

> Patch 流程：createComponent -> 子组件初始化 -> 子组件 render 生成渲染 VNode -> 子组件 patch（遍历子组件渲染 VNode，递归，直到普通节点，执行 insert）
>
> activeInstance 当前激活实例，vm.$vnode 为组件占位 vnode，vm._vnode 为组件渲染 vnode

最终，将组件初始化过程和上篇普通节点初始化过程合并得到以下流程，一个完整 Vue 项目以此循环完成渲染

```
     +----------+
     | new Vue()|
     +----+-----+
          |
     +----v-----+      +--------+    +---------+    +-----------------+
+---->  _init   +------> $mount +----> compile +----> render function +--+
|    +----------+      +--------+    +---------+    +-----------------+  |
|                                                                        |
|                                                                        |
|    +-----+     +-----------+     +-------+ update +-------+    +-------v-------+
|    | DOM <-----+ createElm <-----+ patch <--------+ vnode <----+ createElement +--+
|    +-----+     +-----------+     +-------+        +-------+    +---------------+  |
|                                                                                   |
|                                                                                   |
|    +-------------------------------+  init +-----------+    +--------+   +--------v--------+
+----+createComponentInstanceForVnode<-------+ createElm <----+ CVnode <---+ createComponent |
     +-------------------------------+       +-----------+    +--------+   +-----------------+

```
