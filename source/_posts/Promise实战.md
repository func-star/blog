---
title: Promise实战
date: 2018-06-26 20:04:41
tags:
---

### 一、概述

1. `Promise` 简单来说就是一个对象，他拥有三种状态`pending`、`fulfilled`和`rejected`（过程中、成功和失败）。我们可以简单理解 `Promise` 为一个过程，它不受外界因素干扰，任何其他操作都不能强行改变 `Promise` 的状态。
2.`Promise` 的状态只会从 `pending` 到 `fulfilled` 或者 `pending` 到 `rejected`，简单的说就是一个处理成功的过程和一个处理失败的过程，而且一旦得到结果，状态就不能改变。


### 二、举例说明
注：下面会有很多模拟异步数据返回的形式，这里先封一个公共函数供下面的`demo`使用：
```js
var ajaxDataFn = function(val, delay) {
    return new Promise(function(resolve) {
        setTimeout(function() {
            resolve({ val: val, delay: delay })
        }, delay)
    })
}
```

（1）`Promise` 实例化后会立马执行，状态变化后会触发`then`提前注册好的回调函数（包括成功回调和失败回调）

```js
var promise = new Promise((resolve, reject) => {
    console.log('step1');
    resolve();
});
promise.then(() => {
    console.log('Resolved.');
})
console.log('Hi!')
// step1
// Hi!
// Resolved

```

（2）`.then()`的链式调用写法

（层级较深）

```js
ajaxDataFn('test1', 500).then(function(res) {
    console.log(res.val);
    ajaxDataFn(`${res.val} test2`, 1000).then(function(res) {
        console.log(res.val);
        ajaxDataFn(`${res.val} test3`, 1500).then(function(res) {
            console.log(res.val);
        })
    })
})
// test1
// test1 test2
// test1 test2 test3
```
(建议的写法)

```js
ajaxDataFn('test1', 500).then(function(res) {
    console.log(res.val);
    return ajaxDataFn(`${res.val} test2`, 1000)
}).then(function(res) {
    console.log(res.val);
    return ajaxDataFn(`${res.val} test3`, 1500)
}).then(function(res) {
    console.log(res.val);
})
// test1
// test1 test2
// test1 test2 test3
```
两种写法都能达到逻辑上的目的，但是用这种链式的写法就能很好的避免掉层级过深的问题，逻辑也能更加清晰，偶合度也低。

（3）`.catch()`的使用

demo1:
```js
var promise = new Promise((resolve, reject) => {
    throw new Error('error');
});
promise.catch(err => console.log("error: ", err));
// error:  Error: error
```
demo2:

```js
var  promise = new Promise((resolve, reject) => {
    reject(new Error('error'));
});
promise.catch(err => console.log("error: ", err));
// error:  Error: error
```
demo3:

```js
var promise = new Promise((resolve, reject) => {
    throw new Error('error');
});
promise.catch(err => {
    console.log("error: ", err)
    return new Promise((resolve, reject) => {
        resolve('catchVal')
    })
}).then(res => {
    console.log(res)
});
// error:  Error: error
// catchVal
```

**tip:** `.catch()` 仍然可以返回一个 `Promise` 对象，后面还可以继续 `.then()` 调用


（4）`all()`、`race(`)的使用
可以参考[Promise介绍篇](https://github.com/func-star/promise/wiki/Promise%E4%BB%8B%E7%BB%8D%E7%AF%87)中的demo

（5）`reject()`、`resolve()`的使用
```js
Promise.resolve('test').then(function(res) {
    console.log(res);
})
// test
```
```js
Promise.reject('test').then(null, function(res) {
    console.log(res);
})
// test
```
```js
var p = new Promise(function(resolve) {
    resolve('test');
})

Promise.resolve(p).then(function(res) {
    console.log(res);
})
// test
```
```js
console.log(p === Promise.resolve(p))
// true
```
**tip：** `Promise.resolve()`会返回一个 `Promise` 示例，它的参数可以是一个值也可以是一个 `Promise` 实例，且当参数是 `Promis` 实例时，`Promise.resolve(p) === p`。因此在真实场景中，当使用第三方库时，别人提供的 `Promise` 实例返回，建议都用 `Promise.resolve()` 包装后再使用，这样是最安全的。哪怕别人的库写的有问题也不会影响我们逻辑的正常运行。


### 三、实战应用

先介绍`.then()`的实现简化版原理(可以前往[Promise原理篇](https://github.com/func-star/promise/wiki/Promise%E5%8E%9F%E7%90%86%E7%AF%87)查看)

```js
 this.then = function(onFulfilled) {
    return new Promise(function(resolve) {
        function handle() {

            var returnVal = isFn(onFulfilled) && onFulfilled(promise.value) || promise.value;
            // var returnVal = onFulfilled(promise.value);
            if (isFn(returnVal.then)) {
                returnVal.then(function(res) {
                    resolve(res)
                })
            } else {
                resolve(returnVal);
            }
        }
        if ('PENDING' === promise.status) {
            promise._resolveList.push(handle);
        } else if ('FULFILLED' === promise.status) {
            handle(promise.value)
        }
    })
}
```
================part1================

下面来试一把。。

```js
function test1() {
    console.log('test1');
    return ajaxDataFn('test1', 1000);
}

function test2() {
    console.log('test2');
    return ajaxDataFn('test2', 1000);
}

function test3() {
    console.log('test3');
    return console.log;
}

// 题1
test1().then(function() {
    return test2();
}).then(test3);

// 题2
test1().then(function() {
    test2();
}).then(test3);

// 题3
test1().then(test2())
    .then(test3);

// 题4
test1().then(test2)
    .then(test3);
```

`test1()` 和 `test2()` 均返回 `promises`。。。这四个题有什么区别呢，执行顺序又是怎样的呢？？？？

=====================================

================part2================

```js
ajaxDataFn([1, 2, 3], 500).then(function(res) {
    res.val.forEach(function(item) {
        ajaxDataFn(item, 500).then(function(res) {
            console.log(res)
        });
    })
}).then(function() {
    console.log('done')
})
// done
// {val: 1, delay: 500}
// {val: 2, delay: 500}
// {val: 3, delay: 500}
```
这段代码表达了通过第一个 `Promise` 返回一个数组 `[1,2,3]`，再通过第二个 `Promise` 来将数组的每一个 `item` 输出，最后想输出结束标识。
然而第一个 `Promis`e 返回的实际上是 `undefined` ，也就是说第二个`.then()`根本不会等到 `list` 中所有的`item`项执行完再去执行。

那么`forEach(或者for)`这类的循环又怎么和`Promise`一起来使用呢？我们可以看下上面介绍中提到的将所有的 `list`操作包装成一个新的 `Promise` 去执行。

```js
ajaxDataFn([1, 2, 3], 500).then(function(res) {
    return Promise.all(res.val.map(function(item) {
        return ajaxDataFn(item, 500).then(function(res) {
            console.log(res)
        });
    }))
}).then(function() {
    console.log('done')
})
// {val: 1, delay: 500}
// {val: 2, delay: 500}
// {val: 3, delay: 500}
// done
```
=====================================

================part3================

在 `Promise.all()` 和 `.then()` 聊了那么多点之后，咱进阶的聊一下 `.then()` 的好用的地方

看完上述的那么多例子，可以总结出，在then()函数内部可以作3件事。

1.return 一个`Promise`

2.return 一个同步数据（当然可能是`undefined`）

3.throw一个异常

先来看下第一点：

```js
ajaxDataFn('test1', 500).then(function(res) {
    console.log(res.val);
    return ajaxDataFn('test2', 500)
}).then(function(res) {
    console.log(res.val)
})
// test1
// test2
```
这是最常用的一种用法，但是千万别忘了`return`这个关键字哦！如果没写,那么下一个函数接收的就是`undefined`了，而且也不会等第一个请求结束了才去执行第二个。

看第二点：

```js
var cache = null;
ajaxDataFn('test1', 300).then(function(res) {
    if (cache) {
        console.log('缓存中拉取数据')
        return cache;
    }
    return ajaxDataFn('test2', 300).then(function(res) {
    	console.log('从test2 Promise中拉取数据')
        cache = res;
        return cache;
    });
}).then(function(res) {
    console.log(res.val)
})

ajaxDataFn('test1', 1000).then(function(res) {
    if (cache) {
        console.log('缓存中拉取数据')
        return cache;
    }
    return ajaxDataFn('test2', 500).then(function(res) {
        console.log('从test2 Promise中拉取数据')
        cache = res;
        return cache;
    });
}).then(function(res) {
    console.log(res.val)
})
// 从test2 Promise中拉取数据
// test2
// 缓存中拉取数据
// test2
```
我们可以用将一些不变的请求数据存在本地变量中，然后下次请求来的时候就直接返回变量中存储的值，这样我们可以节省一些请求消耗。

然而这样写存在一个隐患，不知道大家发现没？同步数据就会涉及`undefined`的类型，你很有可能会把`undefined`传到下一个函数，因此在后面加上`catch`去捕获异常。

=====================================

================part4================

前面讲到`then()`的第二个参数`reject`和`catch()`并非完全等价，下面就举例来说明

```js
ajaxDataFn('test1', 300).then(function(res) {
    throw new Error('error');
}, function(res) {
	console.log('reject中捕获到了')
    console.log(res)
}).catch(function(error) {
	console.log('catch中捕获到了')
    console.log(error)
})
// catch中捕获到了
// Error: error
```
```js
ajaxDataFn('test1', 300).then(function(res) {
    throw new Error('error');
}, function(res) {
    console.log('reject中捕获到了')
    console.log(res)
}).then(null, function(res) {
	console.log('下一级reject中捕获到了')
    console.log(res)
})
// 下一级reject中捕获到了
// Error: error
```

在同级别 `resolve` 中的处理错误，`reject` 是获取不到错误信息的，但是 `catch` 能获取到。然而下一级的 `reject` 也是能捕获到错误的。

=====================================

================part5================

前面抛出的问题，现在让我们一起来探讨下，是否都能解决了

```js
test1().then(function() {
    return test2();
}).then(test3);
```
```js
test1
|-----------------|
                  test2(undefined)
                  |------------------|
                                     test3(resultOfTest2)
                                     |------------------|
```
这是典型的 `then()` 中 `return` 另一个 `Promise` 的场景，首先 `test2` 在 `test1` 后执行肯定没问题。
而且下一个函数`test3()`会等前一个`test2()`请求完之后才执行。
```js
test1().then(function() {
    test2();
}).then(test3);
```

```js
test1
|-----------------|
                  test2(undefined)
                  |------------------|
                  test3(undefined)
                  |------------------|
```
前面提到在 `then()` 中想要返回另一个 `Promise` 的时候千万别忘了`return`关键字，这也是 `then()` 的副作用用法。
在`test1().then()`中实际返回的是默认值`undefined`，因此 test3 并不会等到 test2 执行结束后才开始执行。
而 `test2` 在 `test1` 执行完再执行肯定没问题。
```js
test1().then(test2())
    .then(test3);
```
```js
test1
|-----------------|
test2(undefined)
|---------------------------------|
                  test3(resultOfTest1)
                  |------------------|
```
下面的例子详细一些：
```js
function test2() {
    return Promise.resolve('bar');
}
Promise.resolve('foo').then(test2).then(function(result) {
    console.log(result);
});
//bar
Promise.resolve('foo').then(test2()).then(function(result) {
    console.log(result);
});
//fool
```

这个知识点我前面没有介绍，在这里我讲一下（可以参考上面给出的`.then()`原理代码来看）。当`then()`中的参数不是一个函数的时候，内部逻辑会立即执行，也就是说在`.then`注册添加回调函数的时候已经执行了，更细致的说在 `Promise` 的状态还是 `pending` 的时候已经执行了 ,而且不会影响整个链路，很多书上称之为 “Promise穿透”，在这个场景中，`test2` 实际返回的是一个 `Promise` 实例不属于函数，所有在 `test1` 执行完之后立马执行 `test3`，而且 `test2` 不会影响整个进度，所以和`test1`同步进行。

```js
test1().then(test2)
    .then(test3);
```

 ```js
test1
|-----------------|
                  test2(resultOfTest1)
                  |------------------|
                                     test3(resultOfTest2)
                                     |------------------|
```
这里 `test2` 就像是把 `then()` 中的回调函数抽离到一个命名函数中。
这样实际上就和第一种情况一样，`then()`中的回调实际`return`了另一个`Promise`。
```js
function test2Copy() {
    return test2()
}
test1().then(test2Copy).then(test3);
```
