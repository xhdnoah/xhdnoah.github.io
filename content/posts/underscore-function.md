+++
title = "UnderScore: 函数"
date = "2019-03-20"
+++

### 上下文绑定

underscore 提供绑定函数上下文的方法 `_.bind` 和 `_.bindAll`，二者的绑定上下文及执行过程都由内部函数 `executeBound` 负责

```
// 执行绑定后的函数
var executeBound = function(sourceFunc, boundFunc, context, callingContext, args){
  if(!(callingContext instanceof boundFunc))
  	return sourceFunc.apply(context, args)
  var self = baseCreate(sourceFunc, prototype)
  var result = sourceFunc.apply(self, args)
  if(_.isObject(result))
  	return result
  return self
}
```

`executeBound` 也考虑了当绑定后的函数 `boundFunc` 作为构造函数被 `new` 运算的情形，进行了执行上下文的修正。另外为了支持链式调用，所以有以下语句：

```
var result = sourceFunc.apply(self, args)
if(_.isObject(result))
	return result
return self
```

**`_.bind(func, context)`** 将 func 的执行上下文绑定到 context

```
_.bind = function(func, context){
  if(nativeBind && func.bind === nativeBind) return nativeBind.apply(func, slice.call(arguments, 1))
  if(!_.isFunction(func)) throw new TypeError('Bind must be called on a function')
  var args = slice.call(arguments, 2)
  var bound = function(){
    return executeBound(func, bound, context, this, args.concat(slice.call(arguments)))
  }
  return bound
}
```

### 偏函数

偏函数 partial 反映了新函数是原函数的一部分，underscore 的 `_.partial` 返回一个偏函数

**`_.partial(func, ...args)`**：偏应用一个函数 `func`

```
_.partial = restArgs(function(func, boundArgs){
  var placeholder = _.partial.placeholder
  var bound = function(){
    // position 标识当前赋值的 arguments 最新位置
    var position = 0, length = boundArgs.length
    var args = Array(length)
    for(var i = 0;i < length; i++){
      args[i] = boundArgs[i] === placeholder ? arguments[position++] : boundArgs[i]
    }
    while(position < arguments.length) args.push(arguments[position++])
    return executeBound(func, bound, this, this, args)
  }
  return bound
})
```

underscore 提供的偏函数支持传递占位符，占位符用来标识不想被初始化的参数

```
var substract = function(a, b){
  return b - a
}
sub5 = _.partial(subtract, 5)
sub5(20) // => 15

// 通过使用一个 placeholder _,先赋值 b，暂缓对 a 赋值
subFrom20 = _.partial(subtract, _, 20)
subFrom20(5)
// => 15
```



### 缓存

```
function fibonacci(n){
  return n<2 ? n : fibonacci(n-1) + fibonacci(n-2)
}
```

````
fibonacci(5) 递归过程
fibonacci(5) = fibonacci(4) + fibonacci(3)
fibonacci(4) = fibonacci(3) + fibonacci(2)
fibonacci(3) = fibonacci(2) + fibonacci(1)
// ...
````

求解斐波那契数列时，递归过程引起了很多重复计算，考虑将计算结果进行缓存，需要利用到高阶函数和闭包：

```
var fibonacciWithMemo = (function(){
  var memo = {}
  return function fibnocci(n){
  	// 如果尚未建立缓存
    if(!memo[n]){
      memo[n] = n < 2 ? n : fibonacci(n-1) + fibonacci(n-2)
    }
    return memo[n]
  }
}())
```

```
fibonacciWithMemo(5)
memo(5) = memo[4] + memo[3]
memo[4] = memo[3] + memo[2]
memo[3] = memo[2] + memo[1]
// ...
```

 underscore 封装了一个通用的缓存函数创建器 `_.memoize`

**`_.memoize(func, hasher)`** 为函数 func 提供记忆功能， hasher 定义如何获得缓存

```
_.memoize = function(func, hasher){
  var memoize = function(key){
  	// 执行记忆函数时，先获得缓存
  	var cache = memoize.cache
  	// 获得缓存地址
  	var address = '' + (hasher ? hasher.apply(this, arguments) : key)
  	// 如果缓存没有命中，则需要调用函数执行
  	if(!_.has(cache, address)) cache[address] = func.apply(this, arguments)
  	return cache[address]
  }
  // 初始化记忆函数的缓存
  memoize.cache = {}
  return memoize
}
```

`_.memoize` 将缓存绑定到了缓存函数的 `cache` 属性上，在创建一个缓存函数时，除了提供原函数 `func` 用来计算值外，还可以提供 `hasher` 函数来自定义如何获得缓存的位置，不设置 `hasher`，则以缓存函数的参数 `key` 标识缓存位置

```
var fibonacci = _.memoize(function(n){
  return n < 2 ? n : fibonacci(n-1) + fibonacci(n-2)
})
fibonacci(5) // => 5
console.log(fibonacci.cache)

// 传递 hasher
var hasher = function(){
  var n = arguments[0]
  return n
}
var fibonacci = _.memoize(function(n){
  return n < 2 ? n : fibonacci(n-1) + fibonacci(n-2)
}, hasher)

fibonacci(5)
console.log(fibonacci.cache)
```



### throttle & debounce

> 前端开发场景：在页面中，单击"查询"按钮，会通过 ajax 异步获取数据。如果这个查询是耗时的，我们需要控制查询速率防止短时间内频繁请求。

> 电梯超时：把电梯完成一次运送，类比一次函数的执行和响应，假设电梯有两种运行策略 `throttle` 和 `debounce`，超时设定 15s
>
> - throttle 策略电梯，保证如果电梯第一个人进来，15s 后准时运送一次，不等待。
> - debounce 策略电梯，如果有人进电梯，等待 15s，如果又有人进来，15s 等待重新计时，直到 15s 超时，开始运送。

#### 节制型提交

新建一个 wrapper 包裹 `sendQuery` 业务，在 `click` 事件后，不会直接调用 `sendQuery`，而是调用 wrapper。wrapper 中获取当前时间，比较距离上次 `sendQuery` 执行事件的间隔是否有 1s，如果有，那么这次请求立即执行，否则计算剩余等待时间延迟执行该请求，借此控制住 `sendQuery` 的调用频率

为防止变量的全局污染，用一个立即执行函数包裹作用域：

```
var delayedQuery = (function(){
  var previous = 0
  var waiting = 1000
  var sendQuery = function(){
  	// 执行的时候，刷新 previous
  	previous = (new Date()).getTime()
  	console.log("sending query")
	}
  return function(){
   var now = (new Date()).getTime()
   var remain = waiting-(now-previous)
   if(remain<=0){
     sendQuery()
   } else {
     setTimeout(sendQuery,remain)
   }
	}
})()

$("#queryBtn").click(delayedQuery)
```

但是业务代码 `sendQuery` 还是耦合了刷新 `previous` 的逻辑，其次如果每个延迟执行的请求都做这样的包裹就太冗余了。写一个通用函数，将"需要控制调用频率的函数"和"对调用频度的限制频率"传给通用函数，返回一个限制了执行频率的函数

```
function throttle(func, waiting){
  var previos = 0
  // 创建一个 wrapper，解耦 func 和 previous 等变量
  var later = function(){
    previous = (new Date()).getTime()
    func()
  }
  return function(){
    var now = (new Date()).getTime()
    var remain = waiting - (now - previous)
    console.log("need waiting" + remain + "ms")
    if(remain <= 0){
      console.log("immediately")
      later()
    } else {
      console.log("delayed")
      setTimeout(later, remain)
    }
  }
}

var sendQuery = function(){
  console.log("sending query")
}
delayedQuery = throttle(sendQuery,1000)
```

但是存在一个问题，我们只保障了第一次回调和接下来所有回调的间隔执行，而没有保障到各个回调间相互的间隔执行，所以被延后执行的那些请求会再某个相近的时间点同时发生。相比之下，underscore 的实现更健壮

**`_.throttle` "节流阀"**

```
_.throttle = function(func, wait, options){
  // timeout 标识最近一次被追踪的调用
  // context 和 args 缓存 func 执行时需要的上下文，result 缓存 func 执行结果
  var timeout, context, args, result
  var previous = 0
  if(!options) options = {}

  //创建一个延后执行函数包裹住 func 的执行过程，通过闭包访问
  var later = function(){
  	// 若设定了开始边界不执行，上次执行时间始终为 0
    previous = options.leading === false ? 0 : _.now()
    // 清空定时器
    timeout = null
    result = func.apply(context, args)
    if(!timeout) context = args = null
  }
  // 返回一个 throttle 化的函数
  var throttled = function(){
    var now = _.now()
    // 首次执行，如果设定开始边界不执行，将上次执行时间定为当前时间
    if(!previos && options.leading === false) previos = now
    var remaining = wait - (now - previous)
    context = this
    args = arguments
    if(remaining <= 0 || remaining > wait){
      if(timeout){
        clearTimeout(timeout)
        timeout = null
      }
      previous = now
      result = func.apply(context, args)
      if(!timeout) context = args = null
      // 延迟执行不存在，且没有设定结尾边界不执行
    } else if(!timeout && options.trailing !== false){
      timeout = setTimeout(later, remaining)
    }
    return result
  }
  throttled.cancel = function(){
    clearTimeout(timeout)
    previous = 0
    timeout = context = args = null
  }
  return throttled
}
```

**`_.debounce`** 函数去抖

```
_.debounce = function(func, wait, immediate){
  var timeout, args, context, timestamp, result
  var later = function (context, args){
  	// 定时器设置回调 later 方法的触发时间，和连续事件触发的最后一次时间戳的间隔
  	// 即某人进入电梯时间和上一次电梯运送时间的间隔
    var last = _.now() - timestamp
		if(last < wait && last >= 0){
      timeout = setTimeout(later, wait - last)
		} else {
      timeout = null
      if(!immediate){
        result = func.apply(context, args)
        if(!timeout)
        	context = args = null
      }
		}
	}
	return function(){
    context = this
    args = arguments

    timestamp = _.now()
    // 立即触发的两个条件
    var callNow = immediate && !timeout
    if(!timeout)
    	timeout = setTimeout(later, wait)
    if(callNow){
      result = func.apply(context, args)
      解除引用
      context = args = null
    }
    return result
	}
}


```
