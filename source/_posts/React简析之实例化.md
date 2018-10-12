---
title: React简析之实例化
date: 2018-07-16 20:07:19
tags:
---

### 概述
这是一个 `React` 原理解读的系列，将带你一步步实现一个简化版的 `React`。不多说废话，现在开始！！！
代码地址：[v1.0](https://github.com/func-star/mo-react/tree/v1.0)


### `React`的主入口
```js
import { render } from 'react-dom'
...
render(<a>123</a>, document.getElementById('appWrapper'))
```
这行代码大家肯定都非常熟悉，`render()` 函数是 `React` 最基本的方法，是整个 `React` 执行的开端。
我们可以看到 `render` 接收两个参数，一个 `dom` 模版，另一个是模版插入的位置，那么 `render()`  具体做了哪些事情呢？
```js
// ReactMount.js

import ReactInstantiate from './ReactInstantiate'

class ReactMount {
    render(nextElement, container, callback) {
        // 节点实例化
        let instance = new ReactInstantiate(nextElement, null, true)
        console.log(instance)
       // 节点挂载
        let node = instance.mount(null, true)
        container.appendChild(node)
    }
}

export default new ReactMount
```

从代码中我们可以看到 `render()` 先会将接收到的 `dom` 结构进行一个实例化的过程，并生成一个实例对象数据结构`instance`。
接着是对实例对象进行挂载，这一节主要是将 `React` 节点解析成浏览器原生节点，添加合成事件、绑定可识别属性等。这一块我们在下一节会进行讲解。
再接着就是节点插入。

### 下面我们来介绍一下节点实例化到底干了哪些事情

在讲实例化之前，首先我们先了解一下 `React` 节点分为4种节点类型，分别是空节点、文本节点、原生节点（浏览器节点）以及 `React` 节点。
根据这个我们先来定义一个数据字典，如下：
```js
// constant.js

export default {
    VERSION: '0.0.1',
    REACT_NODE: 'reactNode',
    EMPTY_NODE: 'emptyNode',
    TEXT_NODE: 'textNode',
    NATIVE_NODE: 'nativeNode'
}

```
实际上，每一个节点实例都会存储节点的一些信息，包含节点类型、节点的唯一key、节点的dom数据、子节点实例数组。
```js
// ReactInstantiate.js

import reactDom from './ReactDom'
import reactEvents from './ReactEvents'
import ReactUpdater from './ReactUpdater'
import Constant from '../constant'
import Util from '../util'
import ValueData from '../data'

export default class ReactInstantiate {
    constructor(element, key) {
        // react节点
        this.currentElement = element
        // 节点唯一key
        this.key = key
        // 节点类型
        this.nodeType = reactDom.getElementType(element)
        // 节点实例化数据结构
        this.childrenInstance = this.instanceChildren()
    }

    // 递归实例化所有节点，忽略react组件节点
    instanceChildren() {
        if (this.nodeType === Constant.EMPTY_NODE || this.nodeType === Constant.TEXT_NODE || !this.currentElement.props || !this.currentElement.props.children) {
            return
        }

        let child = Util.toArray(this.currentElement.props.children)
        let childrenInstance = []
        // 为每一个节点添加唯一key
        child.forEach((v, i) => {
            let key = null
            if (null !== v && typeof v === 'object' && v.key) {
                key = '__@_' + v.key
            }
            if (Util.isArray(v)) {
                let c = []
                v.forEach((item, index) => {
                    let itemKey = '__$_' + i + '_' + index
                    if (null !== v && typeof v === 'object') {
                        if (!item.key) {
                            console.error(ValueData.keyNeedMsg)
                        } else {
                            itemKey = '__@_' + i + '_' + item.key
                        }
                    }
                    c.push(new ReactInstantiate(item, itemKey))
                })
                childrenInstance.push(c)
            } else {
                childrenInstance.push(new ReactInstantiate(v, key))
            }
        })
        return childrenInstance
    }
}
```

至此，节点的实例化过程已经完成，我们得到一个数据装载完整的`react`实例数据。
这边介绍的代码并不完整，可以前往[v1.0](https://github.com/func-star/mo-react/tree/v1.0)，clone下来本地跑一遍。
