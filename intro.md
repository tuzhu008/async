![Async Logo](https://raw.githubusercontent.com/caolan/async/master/logo/async-logo_readme.jpg)

[![Build Status via Travis CI](https://travis-ci.org/caolan/async.svg?branch=master)](https://travis-ci.org/caolan/async)
[![NPM version](https://img.shields.io/npm/v/async.svg)](https://www.npmjs.com/package/async)
[![Coverage Status](https://coveralls.io/repos/caolan/async/badge.svg?branch=master)](https://coveralls.io/r/caolan/async?branch=master)
[![Join the chat at https://gitter.im/caolan/async](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/caolan/async?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

*Async v1.5.x 文档, 请移步 [这里](https://github.com/caolan/async/blob/v1.5.2/README.md)*


Async 是一个实用程序模块，它为使用异步 JavaScript 提供了直接、强大的函数。尽管最初设计用于与 [Node.js](https://nodejs.org/) 一起使用，使用 `npm install --save async` 进行安装。 它也可以直接在浏览器中使用。

也可以通过下面的方法安装 Async：

- [yarn](https://yarnpkg.com/en/): `yarn add async`
- [bower](http://bower.io/): `bower install async`

Async 提供了大约 70 个函数，其中包括通常的“功能性的”(`map`, `reduce`, `filter`, `each`…)以及一些异步控制流的常见模式(`parallel`, `series`, `waterfall`…)。所有这些函数都假设您遵循 Node.js 约定，提供一个单一的回调函数作为异步函数的最后一个参数 —— 这个回调函数的第一个参数应该为一个错误对象，并且这个对调函数只被调用一次。


## 快速示例

```js
async.map(['file1','file2','file3'], fs.stat, function(err, results) {
    // 结果现在是每个文件的统计数据构成的数组
});

async.filter(['file1','file2','file3'], function(filePath, callback) {
  fs.access(filePath, function(err) {
    callback(null, !err)
  });
}, function(err, results) {
    // 结果现在等于现有文件的数组
});

async.parallel([
    function(callback) { ... },
    function(callback) { ... }
], function(err, results) {
    // 可选的回调
});

async.series([
    function(callback) { ... },
    function(callback) { ... }
]);
```

还有更多可用的函数，所以请查看下面的文档以获得完整的列表。这个模块的目标是提供全面的异步 API，所以如果你觉得有什么缺失，请为它创建一个GitHub的问题。

## 常见的陷阱 [(StackOverflow)](http://stackoverflow.com/questions/tagged/async.js)

### 同步迭代函数 （Synchronous iteration functions）

如果您遇到了像`RangeError: Maximum call stack size exceeded.`这样的错误，或者在使用异步时，您可能会使用同步迭代器。这里说的*同步*，指的是一个函数，它在 javascript 事件循环中调用它的回调，而不需要进行任何 I/O 操作或使用任何计时器。反复调用许多回调会很快地溢出堆栈。如果遇到这个问题，请使用 `async.setImmediate` 延迟回调，以此在下一轮事件循环中启动一个新的调用栈。

This can also arise by accident if you callback early in certain cases:
如果在某些情况下提前回调，这也可能是偶然的:

```js
async.eachSeries(hugeArray, function iteratee(item, callback) {
    if (inCache(item)) {
        callback(null, cache[item]); // 如果许多项目被缓存，将会溢出
    } else {
        doSomeIO(item, callback);
    }
}, function done() {
    //...
});
```

将其更改为:

```js
async.eachSeries(hugeArray, function iteratee(item, callback) {
    if (inCache(item)) {
        async.setImmediate(function() {
            callback(null, cache[item]);
        });
    } else {
        doSomeIO(item, callback);
        //...
    }
});
```

由于性能原因，Async 不反对同步迭代。如果您仍在运行堆栈溢出，您可以按照上面的建议进行延迟，或者使用[`async.ensureAsync`](#ensureAsync)来包装函数。这个函数的本质上是异步的，不存在这个问题，不需要额外的回调延迟。

如果 JavaScript 的事件循环仍然有点模糊，请查看[这篇文章](http://blog.carbonfive.com/2013/10/27/the-javascript-event-loop-explained/)或[这个讨论](http://2014.jsconf.eu/speakers/philip-roberts-what-the-heck-is-the-event-loop-anyway.html)，以获得关于它如何工作的更详细的信息。


### 多次回调

确保在调用回调时总是`return`，否则在许多情况下会导致多次回调和不可预知的行为。

```js
async.waterfall([
    function(callback) {
        getSomething(options, function (err, result) {
            if (err) {
                callback(new Error("failed getting something:" + err.message));
                // 我们应该在这里 return
            }
            // 由于我们没有 return，因此这个回调依然会被调用，`processData`会被调用两次
            callback(null, result);
        });
    },
    processData
], done)
```

每当回调调用不是函数的最后一个语句时，`return callback(err, result)`总是好的做法。

### 使用 ES2017 `async` 函数

无论我们在哪里接受一个 Node式 的回调函数，Async 都接受 `async` 函数。但是，我们不会传递回调，而是使用返回值，并处理任何抛出的承诺拒绝或错误。

```js
async.mapLimit(files, async file => { // <-无回调函数!
    const text = await util.promisify(fs.readFile)(dir + file, 'utf8')
    const body = JSON.parse(text) // <- 这里将自动捕获转换（parse）错误。
    if (!(await checkValidity(body))) {
        throw new Error(`${file} has invalid contents`) // <- 这个错误也会被捕获
    }
    return body // <- 返回一个值
}, (err, contents) => {
    if (err) throw err
    console.log(contents)
})
```

我们只能检测到原生的的 `async `函数，而不是转换的版本(例如，用 Babel)。否则，可以将 `async` 函数包装在`async.asyncify()`中。

### 将上下文绑定到迭代器

这一节实际上是关于 `bind` 的，而不是关于 Async 的。如果您想知道如何让 Async 在给定的上下文中执行迭代器，或者对于为什么另一个库的方法不能作为迭代器而感到困惑，那么研究这个例子:

```js
// 这是一个简单的对象，它有一个(不必要的迂回)求平方的方法
var AsyncSquaringLibrary = {
    squareExponent: 2,
    square: function(number, callback){ 
        var result = Math.pow(number, this.squareExponent);
        setTimeout(function(){
            callback(null, result);
        }, 200);
    }
};

async.map([1, 2, 3], AsyncSquaringLibrary.square, function(err, result) {
    // 结果是 [NaN, NaN, NaN]
    // 错误的原因是 square 函数中的 `this.squareExponent`表达式在 AsyncSquaringLibrary 的上下文中，它是未定义的。
});

async.map([1, 2, 3], AsyncSquaringLibrary.square.bind(AsyncSquaringLibrary), function(err, result) {
    // 结果是 [1, 4, 9]
    // 我们可以在传递迭代器到Async之前， 通过 bind 的帮助，附加一个上下文到迭代器。
    // 现在 square 函数将在 AsyncSquaringLibrary 上下文中被执行，`this.squareExponent`的值也会被正确计算。
});
```

## 下载

可以从[GitHub](https://raw.githubusercontent.com/caolan/async/master/dist/async.min.js) 下载源码。

另外，你可以通过 npm 安装：

```bash
$ npm install --save async
```

也可以使用 Bower:

```bash
$ bower install async
```

然后，可以正常`require()` async :

```js
var async = require("async");
```

或者 require 单独的方法:

```js
var waterfall = require("async/waterfall");
var map = require("async/map");
```

_开发版本:__ [async.js](https://raw.githubusercontent.com/caolan/async/master/dist/async.js) - 29.6kb 未经压缩

### 浏览器中使用

Async 应该在任何 ES5 环境中工作(IE9和以上)。

用法:

```html
<script type="text/javascript" src="async.js"></script>
<script type="text/javascript">

    async.map(data, asyncProcess, function(err, results) {
        alert(results);
    });

</script>
```

Async 的可移植版本，包括 `async.js` 和 `async.min.js`，它们包含在 `/dist` 文件夹中。在[jsDelivr CDN](http://www.jsdelivr.com/projects/async)上也可以找到Async。

### ES 模块


我们还提供作为ES2015模块的集合的Async，在 npm 的`async-es`包中。

```bash
$ npm install --save async-es
```

```js
import waterfall from 'async-es/waterfall';
import async from 'async-es';
```

## 其他库

* [`limiter`](https://www.npmjs.com/package/limiter) 基于每 秒/小时的请求的速率限制的包。
* [`neo-async`](https://www.npmjs.com/package/neo-async) Async 的另一种实现，关注速度。
* [`co-async`](https://www.npmjs.com/package/co-async) 一个由 Async 启发的库，用于以[`co`](https://www.npmjs.com/package/co)和生成器函数使用。
*  [`promise-async`](https://www.npmjs.com/package/promise-async) Async 的一个版本，其中所有的方法都是被 Promise 化的。
