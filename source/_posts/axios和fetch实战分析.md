---
title: axios 和 fetch 实战分析
date: 2018-08-08 20:09:42
tags:
---

### 前言
趁着搭建公司新版 `react` 移动端脚手架的机会，对 `http-client` 库细致了解了一番。`axios` 和 `fetch`是目前比较火热的两款产品，这里就我了解到的一些信息记录一下。

### `fetch api` 使用总结
之前我在 PC 端项目中选择的是 `fetch api`，使用了将近两年左右，这里我将个人的使用体验分享一下。

- `fetch` 默认不会携带 `cookie`， 我们需要手动设置 `credentials` 来支持 `cookie` 携带给服务端。
- `fetch` 的 `response` 返回状态码比较特殊，它把`400`、`500`的返回码都当成成功的返回，我们在使用的时候可能需要特地封装一下。
- `fetch` 没办法取消本次请求，只要你发送了本次请求，后续的情况都是不可控的。
- 另外，我们知道 `fetch` 作为 `AJAX` 的一个替代品，它并不是基于 `XHR` 实现的，所以它并不能检测到请求的进度。

可能或许大概你会拿它和 `$.ajax`、`axios` 这类封装的比较完善的库做对比，而且开始吐槽，这东西怎么哪哪都不如 `axios` 。
这里我们先要确定一下 `fetch api` 的定位，它是一个更加偏底层的库，提供了丰富的 `API` 给开发者调用，可以支持很多的底层功能，但是可能找不到基于场景封装好的功能。
因此，我们在选择 `fetch` 的时候需要考虑我们是不是有这么复杂的需求需要通过 `fetch api` 来支持，用 `axios` 是不是会更省事？

上述介绍了一大堆的 `fetch api` 的缺点，可能大家会认为我对 `fetch api` 有什么偏见... 哈哈😄。实际上我近两年的 `http-client` 库选择的都是 `fetch api`。
还是那句话，没有最好的产品，只有最适合的产品。如果某个产品你用的习惯了，你就咋看咋顺眼。

### `axios` 使用总结
从去年下半年开始，我开始在线上移动端使用 `axios` 。接下来讲一下我个人的使用情况。

刚开始用的时候我也是看了很多的文档资料，毕竟网上把这东西夸的天花乱坠的，到底这东西魅力在哪里，得自己试试才知道呀。

这里先介绍一下我之前对 `axios` 做的分析，具体是不是真的有那么大魅力，我们后面用场景验证。
- `axios` 支持设置全局默认配置，比如项目中基本不会变的根域名、`headers`配置这些，我们都可以初始化设置一发。
- `axios` 拦截器也是一个比较好用的功能，它允许在发 `request` 之前和接收 `response` 之前进行自己的一些逻辑处理。
- 还有一点比较重要的是，这家伙支持请求取消呀，这能一定程度的减少网络请求资源浪费。

以上三点是我比较 care 的特性，其他的我就不罗列介绍了。
下面我们来看一下怎么封一个 `axios` 类(临时写的一个思路，未测试)，让它帮我们的业务更好的赋能。
```js
import axios from 'axios'
import Evnets from 'mona-events'

const CancelToken = axios.CancelToken

class Ajax extends Evnets {
    constructor(props) {
        super(props)
        axios.default.baseURL = 'https://api.monajs.cn'
        this.filter
    }

    get(url, params = {}, canCancel = false, cancelOption) {
        if (canCancel) {
            params.cancelToken = new CancelToken(function executor(c) {
                // executor 函数接收一个 cancel 函数作为参数
                cancelOption = c;
            })
        }
        return axios.get(url, params);
    }

    post(url, params = {}, canCancel = false, cancelOption) {
        if (canCancel) {
            params.cancelToken = new CancelToken(function executor(c) {
                // executor 函数接收一个 cancel 函数作为参数
                cancelOption = c;
            })
        }
        return axios.post(url, params)
    }

    filter() {
        axios.interceptors.request.use(config => {
            // 在发送请求之前做些什么
            return config;
        }, error => {
            // 对请求错误做些什么
            return Promise.reject(error);
        });
        axios.interceptors.response.use(response => {
            // 对响应数据做点什么
            return response;
        }, error => {
            // 对响应错误做点什么
            return Promise.reject(error);
        })
    }
}
```

上面基本上阐述了两个事情：
- `axios` 配合我们的业务场景使用，例如结合服务端的业务返回码，接口请求异常，做统一逻辑处理，规避掉很多冗余代码。
- `axios` 类初始化一些配置信息，在真实使用的时候其他的接口服务层类可以继承这个类，可以改写覆盖这些配置信息，也可以使用默认值。
- `axios` 类配合 `Events` 类完成服务接口监听者模式，更高效的实现接口数据分发，减少掉代码中的回调函数传递。（这个点后续专门开一篇文章详细阐述其使用魅力）

总结下来我的观点是选择哪个 `http-client` 库进行业务开发取决于实际场景，在移动端开发，我个人比较偏向于 `axios` 。它能够帮我们较少成本的上手使用。
