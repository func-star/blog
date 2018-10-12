---
title: Promise介绍
date: 2018-06-26 20:01:10
tags:
---

![1](https://user-images.githubusercontent.com/13312192/38787780-b97c1180-4162-11e8-81a2-346b6ce0b1b3.png)

可以看到`Promise`其实是个构造函数，原型上挂着`catch`和`then`方法，自身拥有`race`、`all`、`reject`、`resolve`等方法。

那既然是构造函数，那我们就直接`new`一个来看一下。

```js
var promise = new Promise(function (resolve, reject) {
    setTimeout(function () {
        console.log('第一个异步请求开始')
        resolve('请求返回');
    })
})
// 第一个异步请求开始
```
我们可以看到，`Promise()`接收一个包含两个参数的函数，两个参数分别为`resolve`和`reject`，对应着成功和失败。而且在实例化一个`Promise`的时候，回调函数会马上执行。

```js
function promise() {
    return new Promise(function (resolve, reject) {
        setTimeout(function () {
            console.log('第一个异步请求开始');
            resolve('成功返回');
        })
    })
}
```
通过这个函数，我们来看下面的`demo`：

```js
promise().then(function(data){
    console.log(data)
})
// 第一个异步请求开始
// 成功返回
```

`第一个异步请求开始`被打印出来肯定都清楚，可是`成功返回`也被打印出来了。前面介绍到 `Promise` 构造函数的原型上挂着两个 `catch` 和 `then` 两个方法，我们可以在 `Promise` 实例上链式调用`.then()`。

`.then()` 可以接收两个回调函数作为参数，第一个对应 `fullfiled(resoleved、成功)` 执行，第二个对应 `rejected(失败)` 执行。并且这两个回调函数能拿到前面 `resolve(arg)` 和 `reject(arg)` 传入的参数。

接下来`reject`同理:

```js
function promise() {
    return new Promise(function(resolve, reject) {
        setTimeout(function() {
            console.log('第一个异步请求开始');
            reject('失败返回');
        })
    })
}
promise().then(function(data) {
    console.log(data)
}, function(data) {
    console.log(data)
})
// 第一个异步请求开始
// 失败返回
```

我们接着来看下面的`demo`：

```js
function promise() {
    return new Promise(function(resolve, reject) {
        setTimeout(function() {
            console.log('第一个异步请求开始');
            resolve('成功返回');
            reject('失败返回');
        }, 1000)
    })
}
promise().then(function(data) {
    console.log(data)
}, function(data) {
    console.log(data)
})
// 第一个异步请求开始
// 成功返回
```

我们可以看到这里的 `reject('失败返回')` 根本没有被触发。实际上，`promise` 的状态只会从 `pending` 到 `resolved` 或者 `pending` 到 `rejected`。

简单的说就是一个处理成功的过程和一个处理失败的过程，而且一旦得到结果，状态就不能改变。

接下来我们来看一下 `Promise`的链式调用，如何将异步代码以同步模式组织起来：

```js
function test1() {
    return new Promise(function(resolve, reject) {
        setTimeout(function() {
            console.log('异步操作1开始');
            resolve('成功返回test1');
        }, 1000)
    });
}

function test2() {
    return new Promise(function(resolve, reject) {
        setTimeout(function() {
            console.log('异步操作2开始');
            resolve('成功返回test2');
        }, 1500)
    });
}

test1().then(function(data) {
    console.log(data)
    return '非Promise' //看这里
}).then(function(data) {
    console.log(data);
    return test2();
}).then(function(data) {
    console.log(data);
})
// 异步操作1开始
// 成功返回test1
// 非Promise
// 异步操作2开始
// 成功返回test2
```

这里我们可以发现 `.then()` 中如果不返回一个 `Promise` 实例而是返回一个值，也能被下一个 `.then()` 方法接收到。

### `.then()`的原理(可以前往[Promise原理篇](https://github.com/func-star/promise/wiki/Promise%E5%8E%9F%E7%90%86%E7%AF%87)查看)

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
除了 `then`， 在 `Promise` 的构造函数的原型中还有一个 `catch()` 方法，这个方法其实很容易理解，`catch` 会将 `Promise` 的执行报错和 `resolve` 执行过程中的报错呈现出来，而 `reject` 无法获取 `resolve` 执行过程中的报错。

接下来让我们来看下`.all()`的用法

```js
function test1() {
    return new Promise(function(resolve, reject) {
        setTimeout(function() {
            console.log('异步操作1开始');
            resolve('成功返回1');
        }, 1000)
    });
}

function test2() {
    return new Promise(function(resolve, reject) {
        setTimeout(function() {
            console.log('异步操作2开始');
            resolve('成功返回2');
        }, 1500)
    });
}

function test3() {
    return new Promise(function(resolve, reject) {
        setTimeout(function() {
            console.log('异步操作3开始');
            resolve('成功返回3');
        }, 1000)
    });
}
Promise.all([
    test1(),
    test2(),
    test3()
]).then(function(results) {
    console.log(results);
}).catch(function(reason) {
    console.log(reason);
})
// 异步操作1开始
// 异步操作2开始
// 异步操作3开始
// Array ["成功返回1", "成功返回2", "成功返回3"]
```

`.all()` 的作用相当于把三个异步操作并成一个来执行，等所有的异步操作都返回后，统一进入到 `.then()`。

那么大家普遍都会有疑问，如果其中一个异步操作`reject`了呢，结果会怎样，现在我把第二个修改下

```js
function test2() {
    return new Promise(function(resolve, reject) {
        setTimeout(function() {
            console.log('异步操作2开始');
            reject('拒绝返回2')
        }, 1500)
    });
}
// 异步操作1开始
// 异步操作2开始
// 异步操作3开始
// 拒绝返回2
```

结果是 `.then()` 中的回调函数只接收了 `rejected` 状态的值而且不再是一个数组。

接下来来看一下 `.race()` 方法，这个方法的用法和 `.all()` 差不多，区别在于它不是当所有异步操作都结束后才进入 `.then()`，而 `.race` 是任意其中一个结束它就进入 `.then()` 方法。
```js
function test1() {
    return new Promise(function(resolve, reject) {
        setTimeout(function() {
            console.log('异步操作1开始');
            resolve('成功返回1');
        }, 1000)
    });
}

function test2() {
    return new Promise(function(resolve, reject) {
        setTimeout(function() {
            console.log('异步操作2开始');
            resolve('成功返回2');
        }, 1500)
    });
}
Promise.race([
    test1(),
    test2()
]).then(function(results) {
    console.log(results);
}).catch(function(reason) {
    console.log(reason);
})
// 异步操作1开始
// 成功返回1
// 异步操作2开始
```
