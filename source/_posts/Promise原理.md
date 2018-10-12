---
title: Promise原理
date: 2018-06-26 20:04:05
tags:
---

在看这篇文章之前建议先看[Promise介绍篇](https://github.com/func-star/promise/wiki/Promise%E4%BB%8B%E7%BB%8D%E7%AF%87)和[Promise实战篇](https://github.com/func-star/promise/wiki/Promise%E5%AE%9E%E6%88%98%E7%AF%87)，对 `promise` 有一定的了解能更高效的理解其原理。

我们通过`demo`来看一下 `Promise` 的一些特性。
```js
new Promise(function(resolve) {
    resolve('B')
}).then(function(res) {
    console.log(res)
})

console.log('A')
// A、B
```
通过前面两篇文章的介绍，我们了解到：
1. `Promise`的参数会立即执行，不受执行状态的影响。
2. `then` 负责添加后续要执行的回调函数。
3. 从程序实现上可以理解为把`then`方法的函数参数添加到一个执行队列，然后在`Promise`的状态从 `pending` 改变为 `fulfilled` 或者 `rejected`时，遍历执行这个操作队列。

接下来我们来实现一下：

### 雏形实现
```js
function Promise(fn) {
    var promise = this;
    promise.callBackList = [];

    this.then = function(fn) {
        this.callBackList.push(fn)
    }

    function resolve(value) {
        promise.callBackList.forEach(function(cb) {
            cb(value)
        })
    }
    fn(resolve)
}
```
接下来我们用我们刚实现的 `Promise` 来验证一把：
```js
new Promise(function(resolve) {
    resolve('B')
}).then(function(res) {
    console.log(res)
})

console.log('A')
// A
```
我们发现这个结果与我们预期的输出结果不一样，我们预期会先输出一个`A`然后输出`B`，然而结果却只输出一个`A`。分析一下上面的 `Promise` 雏形实现，不难发现 `resolve` 会先于 `then` 执行，也就是说 `callBackList` 是一个空数组。

### 支持异步

为了解决这个问题，我们可以为 `resolve` 的执行逻辑添加`setTImeout`，让这段逻辑在`Js`任务队列的末尾执行。

```js
function resolve(value) {
    setTimeout(function() {
        promise.callBackList.forEach(function(cb) {
            cb(value)
        })
    })
}
```

### 支持链式调用

经常用 `Promise` 的同学都知道支持链式调用是 `Promise` 的一个优势，它支持将异步的代码以同步的方式来书写，从而使得代码逻辑清晰已读。实际上我们就是让 `.then()` 函数执行后返回`this`。

```js
this.then = function(fn) {
    this.callBackList.push(fn)
    return this
}
```

### 添加执行状态

在[Promise介绍篇](https://github.com/func-star/promise/wiki/Promise%E4%BB%8B%E7%BB%8D%E7%AF%87)中我们介绍过 `Promise` 有 `pending`、`fulfilled`、`rejected` 三个互斥状态，存在 `pending` 到 `fulfilled` 和 `pending` 到 `rejected` 两种场景，而且都是不可逆的。

下面结合状态来升级一把我们前面的雏形 `Promise`，这里先介绍`resolve` 执行时的状态变化，后续补充`reject`：

```js
function Promise(fn) {
    var promise = this;
    promise._resolveList = [];
    promise.status = 'PENDING';
    promise.value = null;
    this.then = function(onFulfilled) {
        if ('PENDING' === this.status) {
            this._resolveList.push(onFulfilled);
            return this;
        }
        onFulfilled(this.value);
        return this;
    }

    function resolve(value) {
        setTimeout(function() {
            promise.status = 'FULFILLED';
            promise.value = value;
            promise._resolveList.forEach(function(cb) {
                cb(promise.value);
            });
        });
    }
    fn(resolve)
}

```
下面来通过个`demo`来验证一发：
```js
new Promise(function(resolve) {
    resolve('B');
}).then(function(res) {
    console.log(res);
})

console.log('A');
// A 、 B
```

写到这里，简单场景下的 `Promise` 已经实现完毕了。但是在实际应用场景中，`Promise` 大多是串行使用的，会通过很多的 `then`链式调用来拆分一个层级比较深的逻辑。

### 串行实现

先举个简单的例子来说明：
```js
new Promise(function(resolve) {
    resolve('B');
}).then(function(res) {
    console.log(res);
    return 'C'
}).then(function(res) {
    console.log(res);
})

console.log('A');
// A 、 B 、 B
```

通过这个例子，我们发现这输出结果又和我们预期的结果有出入了。我们期望输出 `ABC`，可实际上却输出了 `ABB`。这主要是因为我们的整个 `then` 执行队列中的回调函数都共用了 `resolve` 传进来的同一个 `value`。
在[Promise介绍篇](https://github.com/func-star/promise/wiki/Promise%E4%BB%8B%E7%BB%8D%E7%AF%87)中我们介绍过在`then`中重新返回一个新的 `Promise` 示例，提供给下一个 `then` 的回调函数来使用。

来看一下代码实现：
```js
function Promise(fn) {
    var promise = this;
    promise._resolveList = [];
    promise.status = 'PENDING';
    promise.value = null;

    this.then = function(onFulfilled) {
        return new Promise(function(resolve) {
            function handle() {
                var value = onFulfilled(promise.value);
                resolve(value);
            }
            if ('PENDING' === promise.status) {
                promise._resolveList.push(handle);
            } else if ('FULFILLED' === promise.status) {
                handle(promise.value)
            }
        })
    }

    function resolve(value) {
        setTimeout(function() {
            promise.status = 'FULFILLED';
            promise.value = value;
            promise._resolveList.forEach(function(cb) {
                cb(promise.value);
            });
        });
    }
    fn(resolve)
}
```
我们再用刚次的例子来验证一下：
```js
new Promise(function(resolve) {
    resolve('B');
}).then(function(res) {
    console.log(res);
    return 'C'
}).then(function(res) {
    console.log(res);
})

console.log('A');
// A 、 B 、 C
```
这个例子验证OK了，接下来我们再验证另一种场景，`then` 里面如果返回的是一个 `Promise` 示例又会是什么样的结果呢？
```js
new Promise(function(resolve) {
    resolve('B');
}).then(function(res) {
    console.log(res);
    return new Promise(function(resolve) {
        resolve('C')
    });
}).then(function(res) {
    console.log(res);
})

console.log('A');
// A、B、Promise对象
```
观察输出结果，又和我们的预期有出入了，我们希望输出的是 `C` ，然而输出了一个`Promise`对象，接下来我们来对 `then` 完善这种场景。
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
再来验证一下上面的`demo`：
```js
new Promise(function(resolve) {
    resolve('B');
}).then(function(res) {
    console.log(res);
    return new Promise(function(resolve) {
        resolve('C')
    });
}).then(function(res) {
    console.log(res);
})

console.log('A');
// A、B、C
```

### 处理失败
上面介绍了成功时会将 `pending` 状态置为 `fulfilled` ，失败时会将 `pending` 置为 `rejected`，下面来把失败场景的代码补上：
```js
function Promise(fn) {
    var promise = this;
    promise._resolveList = [];
    promise._rejectList = [];
    promise.status = 'PENDING';
    promise.value = null;
    promise.reason = null;

    this.then = function(onFulfilled, onRejected) {
        return new Promise(function(resolve, reject) {
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

            function errback(reason) {
                reason = isFn(onRejected) && onRejected(promise.reason) || promise.reason;
                reject(reason);
            }
            if ('PENDING' === promise.status) {
                promise._resolveList.push(handle);
                promise._rejectList.push(errback);
            } else if ('FULFILLED' === promise.status) {
                handle(promise.value)
            } else if ('FULFILLED' === promise.status) {
                errback(promise.reason)
            }
        })
    }

    function isFn(fn) {
        return typeof fn === 'function'
    }

    function resolve(value) {
        setTimeout(function() {
            promise.status = 'FULFILLED';
            promise.value = value;
            promise._resolveList.forEach(function(cb) {
                cb(promise.value);
            });
        });
    }

    function reject(reason) {
        setTimeout(function() {
            promise.status = 'REJECTED';
            promise.reason = reason;
            promise._rejectList.forEach(function(cb) {
                cb(promise.reason);
            });
        });
    }
    fn(resolve, reject)
}
```

到这里`Promise`的实现原理就over了，下面来讲一下`Promise.resolve`、`Promise.reject`、`catch`等`api`。

### `Promise.resolve`
```js
Promise.resolve = function(value){
    return new Promise(function(resolve, reject){
        resolve(value)
    })
}
```

### `Promise.reject`
```js
Promise.reject = function(value){
    return Promise(function(resolve, reject){
        reject(value)
    })
}
```

### `catch`
在[Promise介绍篇](https://github.com/func-star/promise/wiki/Promise%E4%BB%8B%E7%BB%8D%E7%AF%87)中讲到过，`reject` 和 `catch` 并非完全等价，`catch`不仅能捕获到 `Promise` 的处理错误，而且还能捕获到 `resolve` 执行过程中的报错信息。实际上 `catch` 是通过在`Promise` 中加了一层`then`而间接的达到这种效果。
```js
this.catch = function(onRejected){
    return this.then(undefined, onRejected)
}
```
