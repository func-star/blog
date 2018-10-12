---
title: React简析之节点挂载
date: 2018-07-16 20:08:17
tags:
---

#### 概述
在 [React简析之实例化（一）](https://github.com/func-star/blog/issues/10) 中，我们将节点进行了实例化，得到了一个数据装载完整的 `React` 实例数据。这一节我们来介绍一下节点挂载的过程。
代码地址：[v1.1](https://github.com/func-star/blog/issues/10)

```js
// 挂载节点
mount(parentNode) {
    if (this.nodeType === Constant.EMPTY_NODE) {
        return null
    }

    if (this.nodeType === Constant.REACT_NODE) {
        let componentObj = new this.currentElement.type(this.currentElement.props)
        // component实例
        this.componentObj = componentObj
        // 获取react render()出来的真实节点信息
        let componentElement = componentObj.render()
        if (!componentElement) {
            return null
        }
        this.componentElement = componentElement
        // 监听setState方法
        this.componentObj.__events.on('stateChange', () => {
            this.receiveComponent()
        })
        this.componentInstance = new ReactInstantiate(this.componentElement, null)
        let node = this.componentInstance.mount(parentNode)
        return node
    }

    if (!this.nativeNode) {
        this.nativeNode = reactDom.create(this.currentElement)
        // 事件绑定
        if (this.nodeType === Constant.NATIVE_NODE && this.currentElement.props) {
            reactEvents.register(this.nativeNode, this.currentElement.props)
        }
        if (parentNode) {
            this.parentNode = parentNode
            parentNode.appendChild(this.nativeNode)
        }
    }

    this.nativeNode['__reactInstance'] = {
        _currentElement: this.currentElement
    }

    this.mountChildren(this.nativeNode)
    return this.nativeNode
}

// 挂载子节点// 挂载节点
mount(parentNode) {
    if (this.nodeType === Constant.EMPTY_NODE) {
        return null
    }

    if (this.nodeType === Constant.REACT_NODE) {
        let componentObj = new this.currentElement.type(this.currentElement.props)
        // component实例
        this.componentObj = componentObj
        // 获取react render()出来的真实节点信息
        let componentElement = componentObj.render()
        if (!componentElement) {
            return null
        }
        this.componentElement = componentElement
        // 监听setState方法
        this.componentObj.__events.on('stateChange', () => {
            this.receiveComponent()
        })
        this.componentInstance = new ReactInstantiate(this.componentElement, null)
        let node = this.componentInstance.mount(parentNode)
        return node
    }

    if (!this.nativeNode) {
        this.nativeNode = reactDom.create(this.currentElement)
        // 事件绑定
        if (this.nodeType === Constant.NATIVE_NODE && this.currentElement.props) {
            reactEvents.register(this.nativeNode, this.currentElement.props)
        }
        if (parentNode) {
            this.parentNode = parentNode
            parentNode.appendChild(this.nativeNode)
        }
    }

    this.nativeNode['__reactInstance'] = {
        _currentElement: this.currentElement
    }

    this.mountChildren(this.nativeNode)
    return this.nativeNode
}

// 挂载子节点
mountChildren(parentNode) {
    if (!this.childrenInstance || this.childrenInstance.length === 0) {
        return
    }
    this.childrenInstance.forEach((v) => {
        if (Util.isArray(v)) {
            v.forEach((child) => {
                child.mount(parentNode)
            })
        } else {
            console.log(parentNode)
            v.mount(parentNode)
        }
    })
}
```

#### 这里注意一个点：
实际上在实例化过程中，我们并不会将 `React` 节点转化成原生节点进行实例化。`React` 节点转化成原生节点这一步是在节点挂载的时候完成的。
在 `React` 节点挂载的过程中，我们会根据实例化好的节点信息`reactDom.create(this.currentElement)` 创建出一个真实的 原生节点（浏览器节点），然后插入 `dom` 结构中去。
如果碰到 `React` 节点，则是先通过 `render()` 方法返回的 `dom` 模版，实例化出一份 节点实例，再进行节点挂载。
依次递归完成子节点的挂载操作。

#### 合成事件绑定、属性绑定（可以先忽略）
在 `React` 的节点属性中，通常以驼峰形式命名，像`style`属性接收的也是 `JSON` 对象形式，与原生节点的属性格式不同。

```js
prefixList = [
    'transform',
    'transition'
];
cssNumber = [
    "column-count",
    "fill-opacity",
    "font-weight",
    "line-height",
    "opacity",
    "order",
    "orphans",
    "widows",
    "z-index",
    "zoom"
];

//将样式对象转化为可使用的样式对象
parseStyleObj(data) {
    let _data = {};
    Object.keys(data).forEach((v) => {
        Util.upperToLine(v)
        let name = Util.upperToLine(v);
        let val = data[v];
        if (typeof(val) == 'number' && this.cssNumber.indexOf(name) < 0) {
            val = val + 'px';
        }
        _data[name] = val;
        if (this.prefixList.indexOf(name) >= 0) {
            _data["-webkit-" + name] = val;
        }
    })
    return _data;
}
styleObjStringify(styleObj) {
    if (!styleObj || Object.keys(styleObj).length == 0) {
        return '';
    }
    return Object.keys(styleObj).map((v) => {
        return v + ":" + styleObj[v];
    }).join(";")
}

//dom信息绑定
parse(node, props) {
    if (!props) {
        return;
    }
    let propKeys = Object.keys(props);

    //className
    if (Util.has(props, 'className')) {
        if (props.className) {
            node.className = props.className;
        }
        Util.arrayDelete(propKeys, "className")
    }

    //style
    if (Util.has(props, 'style')) {
        let style = this.parseStyleObj(props.style);
        node.setAttribute("style", this.styleObjStringify(style));
        Util.arrayDelete(propKeys, "style")
    }

    //defaultValue
    if (Util.has(props, 'defaultValue')) {
        node.setAttribute("value", props.defaultValue);
        Util.arrayDelete(propKeys, "defaultValue")
    }

    //dangerouslySetInnerHTML
    if (Util.has(props, 'dangerouslySetInnerHTML')) {
        if (!props.dangerouslySetInnerHTML || !props.dangerouslySetInnerHTML.__html) {
            return;
        }
        node.innerHTML = props.dangerouslySetInnerHTML.__html;
        Util.arrayDelete(propKeys, "dangerouslySetInnerHTML")
    }

    propKeys.forEach((v) => {
        let val = props[v];
        if (Util.isUndefined(val)) {
            return;
            //val = true;
        }

        if (DOMPropertyConfig.isProperty(v)) {
            let attr = Util.upperToLine(v)
            if (v == 'htmlFor') {
                attr = 'for';
            }
            if (val === false) {
                node.removeAttribute(attr);
                return;
            }
            if (val === true) {
                node.setAttribute(attr, 'true');
            } else {
                node.setAttribute(attr, val);
            }
            return;
        }

        //data-*
        if (/^data-.+/.test(v) || /^aria-.+/.test(v)) {
            node.setAttribute(v, val);
            return;
        }

        //事件绑定
        let e = DOMPropertyConfig.getEventName(v);
        if (e) {
            if (Util.isFun(val)) {
                node.addEventListener(e, (ev) => {
                    val();
                }, false)
            }
        }
    })
}
```
#### `setState` 介绍
在理解 `React` 之前我们都会把 `setState` 当成是一个异步过程，下面见一个例子：
```js
class App extends Component {
	state = { val: 0 }
	increment = () => {
		this.setState({ val: this.state.val + 1 })
		console.log(this.state.val) // 输出的是更新前的val --> 0
	}
	render () {
		return (
			<div onClick={this.increment}>test</div>
		)
	}
}

```
从结果上看，我们很容易把 `setState` 当成是一个异步方法。
其实 `setState` 的“异步”并不是说内部由异步代码实现，其实本身执行的过程和代码都是同步的，只是 `increment`（钩子函数）的调用顺序在更新之前，导致在钩子函数中没法立马拿到更新后的值，形式了所谓的“异步”，当然可以通过第二个参数 `setState(partialState, callback)` 中的`callback`拿到更新后的结果。
那么如何让 `setState` 以同步形式使用呢？

- 用 `setTimeout` 的形式强制改变调用栈
- 用原生事件绑定形式，不使用 `React` 的合成事件

```js
componentDidMount() {
    document.body.addEventListener('click', this.changeValue, false)
 }
```

建议跑一下代码试试：[v1.1](https://github.com/func-star/blog/issues/10)
