+++
title = "UnderScore: 集合"
date = "2019-03-19"
+++

**\_.each**

```js
_.each = _.forEach = function(obj,iteratee, context){
  // 首先要优化回调过程
  iteratee = optimizeCb(iteratee, context)
  var i, length
  if(isArrayLike(obj)){
    for(i = 0, length = obj.length; i < length; i++){
      // 数组迭代回调传入三参数：迭代值，迭代索引，迭代对象
      iteratee(obj[i], i, obj)
    }
  } else {
    var keys = _.keys(obj)
    for (i = 0, length = keys.length; i < length; i++){
      // 对象迭代回调传入三参数：迭代值，迭代 key，迭代对象
      iteratee(obj[keys[i],keys[i],obj)
    }
  }
  // 返回对象自身，以便进行链式构造
}
```

## map-reduce

map reduce 是函数式编程的重要组成，是一种对列表的操作思路。

- **map**（映射）：一个映射过程就是将各个元素按照规则逐个映射为新的元素
- **reduce**（规约）：一个规约过程迭代每个元素按照规则合并到一个目标对象上

### map 在 underscore 中的实现

- 创建一个新列表或元素
- 遍历原列表或原对象的值，用指定的函数 `func` 作用于每个遍历到的元素，输出一个新的元素放入新列表或对象中

```js
_.map = _.collect = function (obj, iteratee, context) {
  iteratee = cb(iteratee, context);
  var keys = !isArrayLike(obj) && _.keys(obj),
    length = (keys || obj).length,
    results = Array(length); // 定长初始化数组
  for (var index = 0; index < length; index++) {
    var currentKey = keys ? keys[index] : index;
    results[index] = iteratee(obj[currentKey], currentKey, obj);
  }
  return results;
};
```

### reduce 在 underscore 中的实现

underscore 创建了一个内部函数 `createReducer` 用来创建 reduce 函数，完成：

- 区分 reduce 的方向 `dir`，是从序列开端开始做规约过程，还是从序列末端开始做
- 判断用户在使用 `_.reduce` 或 `_.reduceRight` 时，是否传入了第三个参数，即是否传入了规约起点，判断结果由 `initial` 变量标识

对于一个 reduce 函数，其执行过程大致如下：

- 设置一个变量 `memo` 用以缓存当前规约过程的结果，如果用户未初始化 `memo`，则 `memo` 为序列的一个参数。遍历当前集合，对最近迭代到的元素按传入的 `func` 进行规约操作，刷新 `memo`
- 规约过程完成，返回 `memo`

```js
var createReduce = function (dir) {
  var reducer = function (obj, iteratee, memo, initial) {
    var keys = !isArrayLike(obj) && _.keys(obj),
      length = (keys || obj).length,
      index = dir > 0 ? 0 : length - 1;
    // 如果 reduce 没有初始化 memo，则默认为首个元素（从左开始则为第一个元素，从右则为最后一个元素）
    if (!initial) {
      memo = obj[keys ? keys[index] : index];
      index += dir;
    }
    for (; index >= 0 && index < length; index += dir) {
      // 执行 reduce 回调，刷新当前值
      memo = iteratee(memo, obj[currentKey], currentKey, obj);
    }
    return memo;
  };
  return function (obj, iteratee, memo, context) {
    // 如果参数正常，则代表已经初始化了 memo
    var initial = arguments.length >= 3;
    // 所有传入的回调都要通过 optimizeCb 进行优化
    // reducer 因为引入累加器，所以优化函数的第三个参数传入了 4
    // 这样，新的迭代回调第一个参数就是当前的累加结果
    // _.reduce([1,2,3],function(prev,current){})
    return reducer(obj, optimizeCb(iteratee, context, 4), memo, initial);
  };
};
```

最终，，underscore 暴露给了两个方向的 reduce API 给用户：

```js
// 从左到右规约
_.reduce = _.fold1 = _.inject = createReduce(1);
// 由右至左规约
_.reduceRight = _.foldr = createReduce(-1);
```

### 查询

对于元素位置查询，underscore 提供以下 API：

- `_.indexOf`
- `_.lastIndexOf`
- `_.findIndex`
- `_.findLastIndex`
- `_.sortedIndex`

> `_.indexOf` 及 `_.lastIndexOf` 只支持对于数组元素的搜索

对于元素查询。underscroe 提供以下 API：

- `_.find` = `_.detect`
- `_.findWhere`
- `_.where`

如果集合是对象，即集合是键值对构成的，则提供以下 API

- `_findKey`
- `_.pluck`

对于判断元素是否存在，underscore 提供以下 API

- `_.contains`

### `createIndexFinder`

underscore 通过内置的工厂函数 `createIndexFinder` 创建一个索引查询器，`_.indexOf` 及 `_.lastIndexOf` 正是由该函数创建

`createIndexFinder(dir, predicateFind, sortedIndex)` 接收三个参数

- dir：查询方向，`_.indexOf` 正向查询，`_lastIndexOf` 反向查询
- predicateFind：真值检测函数，该函数只有在查询元素不是数字时（`NaN`) 才会使用
- sortedIndex：有序数组的索引获得函数，如果设置了该参数，将假定数组已经有序，从而更加高效地通过针对有序数组的查询函数（比如二分查找）来优化查询性能

```js
var createIndexFinder = function(dir, predicateFind, sortedIndex){
  return function (array, item, idx){
    var i = 0, length = getLength(array)
    // 如果 idx 为 Number，则规定查找位置的起始点
    if(tyepof idx == 'number'){
      if(dir > 0){
          // 正向查找，重置查找起始点
        i = idx >= 0 ? idx : Math.max(idx + length , i)
      } else {
          // 反向查找，重置 length 值
        length = idx >= 0 ? Math.min(idx + 1, length) : idx + length + 1
      } else if(sortedIndex && idx && length){
        // 能用二分查找加速的条件
        idx = sortedIndex(array, item)
        return array[index] === item ? idx : -1
      }
      // 特判，如果查找元素是 NaN （if item !== item then item => NaN
      if(item !== item){
        idx = predicateFind(slice.call(array, i, length), _.isNaN)
        return idx >= 0 ? idx + i : -1
      }
      // O(n) 遍历查找，特判排除了 NaN，可以放心 === 判断相等
      for(idx = dir > 0 ? i : length - 1; idx >= 0 && idx < length; idx += dir){
        if (array[idx] === item) return idx
      }
      return -1
    }
  }
}
```

**`_.indexOf(array,item,sorted)`**
```js
_.indexOf = createIndexFinder(1,_.findIndex,_.sortedIndex)
```
**`_.lastIndexOf(array,item,sorted)`**
```js
_.lastIndexOf = createIndexFinder(-1,_.findLastIndex)
```

**`_.sortedIndex(array, obj, iteratee)`** 使用二分法查找

> 根据比较条件 iteratee 查询 obj 在 array 中的位置

```js
_.sortedIndex = function (array, obj, iteratee, context) {
  iteratee = cb(iteratee, context, 1);
  var value = iteratee(obj);
  var low = 0,
    high = getLength(array);
  while (low < high) {
    var mid = Math.floor((low + high) / 2);
    if (iteratee(array[mid]) < value) low = mid + 1;
    else high = mid;
  }
  return low;
};
```

**`createPredicateIndexFinder`**

除了 `createIndexFinder`，underscore 还内置了一个 `createPredicateIndexFinder` 的工厂函数，不仅能查询直接量在集合中的位置，也通过一个真值检测函数查找位置

```js
var createPredicateIndexFinder = function (dir) {
  return function (array, predicate, context) {
    predicate = cb(predicate, context);
    var length = getLength(array);
    var index = dir > 0 ? 0 : length - 1;
    for (; index > 0 && index < length; index += dir) {
      if (predicate(array[index], index, array)) return index;
    }
    return -1;
  };
};
```

**`_.findIndex(array, predicate)`** 根据条件 `predicate` ，查询元素在 `array` 中出现的位置

`_.findIndex = createPredicateIndexFinder(1)`

用例：

```js
// 下面调用不会返回3，因为 12 会被修正为 _.property(12)
_.findIndex([4, 6, 8, 12], 12);
// => -1

_.findIndex([4, 6, 8, 12], function (value) {
  return value === 0;
}); // => 3
```

**`_.findLastIndex(array, predicate)`**

`_.findLastIndex = createPredicateIndexFinder(-1)`

**`_.findKey(obj, predicate)`**：返回 obj 中第一个满足条件的 `predicate` 的 key

```js
_.findKey = function (obj, predicate, context) {
  predicate = cb(predicate, context);
  var keys = _.keys(obj),
    key;
  for (var i = 0, length = keys.length; i < length; i++) {
    key = keys[i];
    if (predicate(obj[key], key, obj)) return key;
  }
};
```

用例：

```js
var student = {
  name: "xyz",
  age: 18,
};
_.findKey(student, function (value, key, obj) {
  return value === 18;
}); // => "age"
```

**`_.find(obj, predicate) = _.detect`**：obj 中满足条件 predicate 的元素

```js
_.find = _.detect = function (obj, predicate, context) {
  var keyFinder = isArrayLike(obj) ? _.findIndex : _.findKey;
  var key = keyFinder(obj, predicate, context);
  if (key !== void 0 && key !== -1) return obj[key];
};
```

`_.find` 既能检索数组（利用 `_.findIndex` 先确定元素下标），又能检索对象（利用 `_findKey` 先确定 key）

### where

在 SQL 中，我们通常利用到 `where` 关键字：

`select * from users where password="123456" and username="xyz"`

在 JavaScript 中想模拟一个 `where` 函数，就需要为其传递两个参数：

- `table`：集合 `users`
- `attrs`：属性匹配列表 `{password: '12345',username='xyz'}`

`where` 的核心过程依然是集合过滤，所以 `Array.prototype.filter` 将是 `where` 的核心：

```js
function where(table, attrs) {
  // 遍历 table
  return table.filter(function (elem) {
    // 观察当前遍历到的对象是否满足 where 条件
    return Object.keys(attrs).every(function (attr) {
      return elem[attr] === attrs[attr];
    });
  });
}

// 测试
var users = [
  { name: "wxj", age: 18, sex: "male" },
  { name: "zxy", age: 18, sex: "male" },
  { name: "zhangsan", age: 14, sex: "famale" },
];
var ret = where(users, { age: 18, sex: "male" });
// => [
// {name: 'wxj', age: 18, sex: 'male'},
// {name: 'zxy', age: 18, sex: 'male'},
//]
```

**`_.where(obj, attrs)`**：获得满足 `attrs` 的元素

```js
    _.where = function(obj. attrs){
      return _.filter(obj, _matcher(attrs))
    }
```

`_.where` 的实现十分简洁，利用到了 `_.filter` 进行迭代，真值检测函数用到了 `_.matcher`

**`_.matcher(attrs)`**：返回一个校验过程，用以检验对象属性是否匹配给定的属性列表

```js
_.matcher = _.matches = function (attrs) {
  attrs = _.extendOwn({}, attrs);
  return function (obj) {
    return _.isMatch(obj, attrs);
  };
};
```

`_.matcher` 接受传入的属性列表 `attrs`，最终返回一个校验过程，通过`_.isMatch` 来校验 `obj` 中的属性的是否与 `attrs` 的属性相匹配

**`_.isMatch(obj, attrs)`**

```js
_.isMatch = function (object, attrs) {
  var keys = _.keys(attrs),
    length = keys.length;
  if (object == null) return !length;
  var obj = Object(object);
  for (var i = 0; i < length; i++) {
    var key = keys[i];
    // 一旦遇到 value 不等，或 attrs 中的 key 不在 obj 中，立即返回 false
    if (attrs[key] !== obj[key] || !(key in obj)) return false;
  }
  return true;
};
```

**`_.findWhere(obj, attrs)`**：与 `where` 类似，但只返回第一条查询到的记录

```js
_.findWhere = function (obj, attrs) {
  return _.find(obj, _.matcher(attrs));
};
```

**`_.contains(obj, item, fromIndex)=._includes=_.include`**：判断 obj 是否包含 item，可设置查询起点

```js
_.contains = _.includes = _.include = function(obj, item, fromIndex, guard){
  if(!isArrayLike(obj) obj = _.values(obj))
  if(typeof fromIndex != 'number' || guard) fromIndex = 0
  return _.indexOf(obj, item, fromIndex) >= 0
}
```

## underscore 模拟一段 SQL

> `SELECT username, id FROM users where age<30 group by sex;`

- `_.groupBy`
- `_.indexBy`
- `_.countBy`
- `_.partition`

以上函数由内部函数 `group` 返回创建

### `group(behavior, partition)`

- `behavior` 分类规则，包括`_.groupBy,_.indexBy,_.countBy`
- `partition` 是否将集合一分为二

```js
var group = function (behavior, partition) {
  return function (obj, iteratee, context) {
    var result = partition ? [[], []] : {};
    iteratee = cb(iteratee, context);
    _.each(obj, function (value, index) {
      var key = iteratee(value, index, obj);
      behavior(result, value, key);
    });
    return result;
  };
};
```

**`_.groupBy(obj, iteratee)`** 对 obj 按照 iteratee 分组

当 `iteratee` 确定了一个分组，`_.groupBy` ：

- 如果分组结果中存在该分组，将元素追加进该分组
- 否则新建一个分组，并将元素放入

```js
_.groupBy = group(function(result, value, key){
  if(_.has(result, key)) result[key].puah(value) else result[key] = [value]
})
```

**`_.indexBy(obj, iteratee)`** 对 obj 按照 iteratee 进行索引

```js
_.indexBy = group(function(result, value, key){
  result[key] = value
})
```

**`_.countBy(obj, iteratee)`** 对 obj 按照 iteratee 进行计数

```js
_.countBy = group(function (result, value, key) {
  if (_.has(result, key)) result[key]++;
  else result[key] = 1;
});
```

用例：

```js
_.countBy([1, 2, 3, 4, 5], function (num) {
  return num % 2 == 0 ? "even" : "odd";
});
// => {odd: 3, even: 2}
```

**`_.partition(obj, iteratee)`** 将 obj 按照 iteratee 进行分组

```js
_.partition = group(function (result, value, pass) {
  // 分组后的行为
  result[pass ? 0 : 1].push(value);
}, true);
```

用例：

```js
_.partition([0, 1, 2, 3, 4, 5], function (num) {
  return num % 2 !== 0;
});
// => [[1,3,5],[0,2,4]]
```
