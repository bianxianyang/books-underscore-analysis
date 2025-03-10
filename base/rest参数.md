# rest 参数

什么是 rest 参数，就是自由参数，松散参数，这里的自由和松散都是值得参数个数是随意的，与之对应的是 -- **固定参数**。

## 从一个加法器开始说起

现在，我们完成一个函数，该函数支持对两个数进行求和，并将结果返回。

```js
function add(a,b){
    return a+b;
}
```

但我们想对更多的数求和呢？那我们首先想到用数组来传递。

```js
function add(numbers){
  return _.reduce(numbers, function(accum,current){
    return accum+current;
  },0);
}
```

或者直接利用 JavaScript 中的 `arguments`:

```js
function add(){
  var numbers = Array.prototype.slice.call(arguments);
  return _.reduce(numbers, function(accum,current){
    return accum+current;
  },0);
}
add(4,3,4,1,1); // => 13
```

现在，我们获得了一个更加自由的加法函数。但是，如果现在的需求变为必须传递至少一个数到加法器呢？

```js
function add(a){
  var rest = Array.prototype.slice.call(arguments, 1);
  return _.reduce(rest,function(accum, current){
    return accum+current;
  },a);
}
add(2,3,4,5); // => 14
```

在这个 `add` 实现中，我们已经开始有了 rest 参数的雏形，除了自由和松散，rest 还有一层意思，就是他的字面意思 -- **剩余**，所以在许多语言环境中，rest 参数从最后一个形参开始，表示剩余的参数。

## 更理想的方式

然而最后一个 `add` 函数还是把对 rest 参数的获取耦合到了 `add` 的执行逻辑中，同时，这样做还会引起歧义，因为在 add 函数的使用者看来，`add` 函数似乎只需要一个参数 `a`。 而在 python，java 等语言中，rest 参数是需要显示声明的，这种生命能让函数调用者知道哪些参数是 rest 参数，比如 python 中通过 `*` 标识 rest 参数：

```python
def add(a,*numbers):
    sum = a
    for n in numbers:
        sum = sum + n * n
    return sum
```

所以，更理想的方式是，提供一个更直观的方式让开发者知道哪个参数是 rest 参数，比如，现在有一个函数，其支持 rest 参数，那么我们总是假定这类函数的最后一个参数是 rest 参数, 为此，我们需要创建一个工厂函数，他接受一个现有的函数，包装该函数，使之支持 rest 参数：

```js
function add(a, rest){
  return _.reduce(rest,function(accum, current){
    return accum+current;
  },a);
}

function genRestFunc(func) {
  // 新返回的函数支持rest参数
  return function(){
    // 获得形参个数
    var argLength = func.length;
    // rest参数的起始位置为最后一个形参位置
    var startIndex = argLength-1;
    // 最终需要的参数数组
    var args = Array(argLength);
    // 设置rest参数
    var rest = Array.prototype.slice.call(arguments,startIndex);
    // 设置最终调用时需要的参数
    for(var i=0;i<startIndex;i++) {
      args[i] = arguments[i]
    }
    args[startIndex] = rest;
    // => args:[a,b,c,d,[rest[0],rest[1],rest[2]] ]
    return func.apply(this, args);
  }
}
addWithRest = genRestFunc(add);

addWithRest(1,2,3,4); // => 10
```

> 记住，在 JavaScript 中，函数也是对象，并且我们能够通过函数对象的 `length` 属性获得其形参个数

最后，我们来看一下 underscore 的官方实现，他暴露了一个 `_.restArgs` 函数，通过给该函数传递一个 `func` 参数，能够使得 `func` 支持 rest 参数：

```js
/**
 * 一个包装器，包装函数func，使之支持rest参数
 * @param func 需要rest参数的函数
 * @param startIndex 从哪里开始标识rest参数, 如果不传递, 默认最后一个参数为rest参数
 * @returns {Function} 返回一个具有rest参数的函数
 */
var restArgs = function (func, startIndex) {
    // rest参数从哪里开始,如果没有,则默认视函数最后一个参数为rest参数
    // 注意, 函数对象的length属性, 揭示了函数的形参个数
    /*
     ex: function add(a,b) {return a+b;}
     console.log(add.length;) // 2
     */
    startIndex = startIndex == null ? func.length - 1 : +startIndex;
    // 返回一个支持rest参数的函数
    return function () {
        // 校正参数, 以免出现负值情况
        var length = Math.max(arguments.length - startIndex, 0);
        // 为rest参数开辟数组存放
        var rest = Array(length);
        // 假设参数从2个开始: func(a,b,*rest)
        // 调用: func(1,2,3,4,5); 实际的调用是:func.call(this, 1,2, [3,4,5]);
        for (var index = 0; index < length; index++) {
            rest[index] = arguments[index + startIndex];
        }
        // 根据rest参数不同, 分情况调用函数, 需要注意的是, rest参数总是最后一个参数, 否则会有歧义
        switch (startIndex) {
            case 0:
                // call的参数一个个传
                return func.call(this, rest);
            case 1:
                return func.call(this, arguments[0], rest);
            case 2:
                return func.call(this, arguments[0], arguments[1], rest);
        }
        // 如果不是上面三种情况, 而是更通用的
        // (应该是作者写着写着发现这个switch case可能越写越长, 就用了apply)
        var args = Array(startIndex + 1);
        // 先拿到前面参数
        for (index = 0; index < startIndex; index++) {
            args[index] = arguments[index];
        }
        // 拼接上剩余参数
        args[startIndex] = rest;
        return func.apply(this, args);
    };
};

// 别名
 _.restArgs = restArgs;
```

测试一下

```js
function add(a, rest){
  return _.reduce(rest,function(accum, current){
    return accum+current;
  },a);
}

var addWithRest = _.restArgs(add);
addWithRest(1,2,3,4); // => 10
```

> 注意，`restArgs` 函数也是 underscore 最新的 master 分支上才支持的，1.8.3 版本不具备这个功能。

### ES6 中的 rest

现在，最新的 [ES6 标准](http://ariya.ofilabs.com/2013/03/es6-and-rest-parameter.html) 已经能够支持 rest 参数，他的用法如下：

```js
function f(x, ...y) {
  // y is an Array
  return x * y.length;
}
f(3, "hello", true) == 6
```

> 所以如果你的项目能够用到 ES6 了，就用 ES6 的写法吧，毕竟他是标准。
