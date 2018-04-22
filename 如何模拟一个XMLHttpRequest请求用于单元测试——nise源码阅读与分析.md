# 概述

在我们进行单元测试的过程中，如果我们需要对一些HTTP接口进行相关的业务测试，那么我们就需要来模拟HTTP请求的发送与响应，否则我们就无法完成测试的闭环。

目前，有许许多多的测试框架都提供了模拟HTTP请求相关的一些流程功能，我们在这边文章中将会讲到的，就是我们在上一篇关于单元测试的博客[提高代码质量——使用Jest和Sinon给已有的代码添加单元测试](https://juejin.im/post/5ad75f16f265da50602c597b)中提到的Sinon中引用的HTTP模拟框架[nise](https://github.com/sinonjs/nise)。

本文的目标是让读者能够通过这篇文章，知道一个成熟的测试框架是如何来模拟一个HTTP的实现，并且与业务代码进行结合，辅助进行测试。本文内容相对较为简单，基本没有难度，作为一个知识面扩充建议读者快速略读。

通过本文，你可以了解以下内容：

- nise是什么？
- nise的设计思路是怎么样？
- nise是如何与业务代码结合，辅助测试？

# nise是什么

> fake XHR and Server.

nise在Github上面的介绍很简单，虽然只有四个单词，但是却很精确的说明了这个库的含义——构造一个模拟的XHR和Server对象，用来替换原生的对象用来满足测试需求。

它是[Sinon.js](http://sinonjs.org/)的一部分，用来处理HTTP相关测试问题。

该库提供了替换原生的XHR对象和Server相关的接口，但是我们在本文中只介绍关于XHR部分，也就是浏览器中的XHR对象的替换。该部分位于仓库中`/lib/fake-xhr/index.js`中，下文中提到的nise如果没有特别注明，均表示nise中的XHR。

# nise的设计思路是怎么样的

## nise的API接口与使用方法

想要了解nise的设计思路，我们就需要先看下nise的使用方法。

目前，nise提供了以下三个API接口：

```javascript
module.exports = {
    xhr: sinonXhr, // 用来存储原来的XHR对象和一些环境判断属性
    FakeXMLHttpRequest: FakeXMLHttpRequest, // XHR对象构造函数
    useFakeXMLHttpRequest: useFakeXMLHttpRequest //调用后，使用fake XHR对象替换全局，并返回一个带有restore方法的fake XHR对象构造函数
};
```

我们在使用时，只需调用`userFakeXMLHttpRequest`方法，即可将原生的XHR对象替换成nise提供的XHR对象。在测试完成后，我们再调用返回的`restore`方法，这样我们就恢复了原生的XHR对象。

返回的模拟HXR对象还有部分API接口可以调用，这部分我们将在下一节——nise结构中进行介绍。

## nise结构

### 构造函数——FakeXmlHttpRequest

```javascript
// 构造函数，用来存储请求相关的数据如请求状态、请求头等
function FakeXMLHttpRequest(config) {
    EventTargetHandler.call(this);
    this.readyState = FakeXMLHttpRequest.UNSENT; // 原生属性，用来标识请求状态
    this.requestHeaders = {}; // 记录请求headers属性
    this.requestBody = null; // 记录请求body属性
    this.status = 0;
    this.statusText = "";
    this.upload = new EventTargetHandler(); // 上传事件属性
    this.responseType = ""; // 响应类型属性
    this.response = ""; // 响应内容属性
    this.logError = configureLogError(config);

    if (sinonXhr.supportsTimeout) {
        this.timeout = 0;
    }

    if (sinonXhr.supportsCORS) {
        this.withCredentials = false;
    }

    if (typeof FakeXMLHttpRequest.onCreate === "function") {
        FakeXMLHttpRequest.onCreate(this);
    }
}

FakeXMLHttpRequest.useFilters = false;

FakeXMLHttpRequest.addFilter = function addFilter(fn) {} // 增加过滤函数

FakeXMLHttpRequest.defake = function defake(fakeXhr, xhrArgs) {} // 将常用事件如open、send等XHR的方法绑定到模拟的XHR对象上

FakeXMLHttpRequest.parseXML = function parseXML(text) {} // 解析XML

extend(FakeXMLHttpRequest.prototype, sinonEvent.EventTarget, {
    open: function open(method, url, async, username, password) {} // XHR原生方法模拟
    
    readyStateChange: function readyStateChange(state) {} // XHR原生方法模拟
    
    setRequestHeader: function setRequestHeader(header, value) {} // 设置请求header并保存到requestHeaders属性中
    
    setStatus: function setStatus(status) {} // 设置status并保存到status属性中
    
    setResponseHeaders: function setResponseHeaders(headers) {} // 设置响应headers并跟随callback一起返回
    
    send: function send(data) {} // XHR原生方法模拟
    
    abort: function abort() {} // 终止HTTP请求
    
    error: function () {} // XHR原生方法模拟
    
    triggerTimeout: function triggerTimeout() {} // 触发超时
    
    getResponseHeader: function getResponseHeader(header) {} // 获取响应header
    
    getAllResponseHeaders: function getAllResponseHeaders() {} // 获取全部的响应headers
    
    setResponseBody: function setResponseBody(body) {} // 设置响应内容
    
    respond: function respond(status, headers, body) {} // 触发请求的callback函数
    
    uploadProgress: function uploadProgress(progressEventRaw) {} // 上传进度触发事件
    
    downloadProgress: function downloadProgress(progressEventRaw) {} // 下载进度触发事件
    
    uploadError: function uploadError(error) {} // 上传失败触发事件
    
    overrideMimeType: function overrideMimeType(type) {} // 覆盖mineType
});
```

# nise是如何与业务代码结合，辅助测试

通过上面的源码介绍我们可以知道：nise是通过完全模拟一个模拟的XHR对象，然后再使用这个模拟的XHR对象来替换全局的XHR对象。

而我们在进行HTTP相关测试时，参数是由我们传入的，因此不需要进行验证。所以我们最终需要验证的其实是callback中的处理逻辑和结果。因此，我们可以通过以下一个示例来看下它如何与业务代码进行结合。这个示例是在上一篇博客中出现过的示例：

```javascript
test('user', () => {
    let callback = jest.fn();

    HTTPCommon.deleteRemoteSession({
        data: {},
        success: callback
    });

    expect(requests.length).toBe(1);

    requests[0].respond(200, {"Content-Type": 'application/json'}, 'hjava'); // 模拟返回值

    expect(callback.mock.calls[0][0]).toBe('hjava');
});
```

通过`respond`这个方法，fakeXMLHttpRequest对象触发了callback函数，并且将指定数据传递到业务代码中。因此，我们能够通过callback相关的业务逻辑来判断我们的逻辑是否正常。

# 总结

nise通过一个非常常规的方法——模拟一个XHR对象并且实现XHR对象的所有功能来完成针对HTTP请求进行记录的功能。我们再通过nise记录的数据，组合其他的单元测试框架来对业务代码进行测试。

nise的源码只有600余行，而且非常简单易懂。我将原有代码folk一份并加上了部分注释，有兴趣的同学可以看看，具体地址见[此处](https://github.com/HJava/nise/blob/master/lib/fake-xhr/index.js)。


# 附录

- [Sinon.js](http://sinonjs.org/)
- [nise](https://github.com/sinonjs/nise)
- [我folk的nise](https://github.com/HJava/nise/blob/master/lib/fake-xhr/index.js)


