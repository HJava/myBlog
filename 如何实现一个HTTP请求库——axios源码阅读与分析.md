# 概述

在前端开发过程中，我们经常会遇到需要发送异步请求的情况。而使用一个功能齐全，接口完善的HTTP请求库，能够在很大程度上减少我们的开发成本，提高我们的开发效率。

axios是一个在近些年来非常火的一个HTTP请求库，目前在[GitHub](https://github.com/axios/axios)中已经拥有了超过40K的star，受到了各位大佬的推荐。

今天，我们就来看下，axios到底是如何设计的，其中又有哪些值得我们学习的地方。我在写这边文章时，axios的版本为0.18.0。我们就以这个版本的代码为例，来进行具体的源码阅读和分析。当前axios所有源码文件都在`lib`文件夹中，因此我们下文中提到的路径均是指`lib`文件夹中的路径。

本文的主要内容有：

- 如何使用axios
- axios的核心模块是如何设计与实现的（请求、拦截器、撤回）
- axios的设计有什么值得借鉴的地方

# 如何使用axios

想要了解axios的设计，我们首先需要来看下axios是如何使用的。我们通过一个简单示例来介绍以下axios的API。

## 发送请求

```javascript
axios({
  method:'get',
  url:'http://bit.ly/2mTM3nY',
  responseType:'stream'
})
  .then(function(response) {
  response.data.pipe(fs.createWriteStream('ada_lovelace.jpg'))
});
```

这是一个官方的API示例。从上面的代码中我们可以看到，axios的用法与jQuery的ajax很相似，都是通过返回一个Promise（也可以通过success的callback，不过建议使用Promise或者await）来继续后面的操作。

这个代码示例很简单，我就不过多赘述了，下面让我们来看下如何添加一个过滤器函数。

## 增加拦截器（Interceptors）函数

```javascript
// 增加一个请求拦截器，注意是2个函数，一个处理成功，一个处理失败，后面会说明这种情况的原因
axios.interceptors.request.use(function (config) {
    // 请求发送前处理
    return config;
  }, function (error) {
    // 请求错误后处理
    return Promise.reject(error);
  });

// 增加一个响应拦截器
axios.interceptors.response.use(function (response) {
    // 针对响应数据进行处理
    return response;
  }, function (error) {
    // 响应错误后处理
    return Promise.reject(error);
  });
```

通过上面的示例我们可以知道：在请求发送前，我们可以针对请求的config参数进行数据处理；而在请求响应后，我们也能针对返回的数据进行特定的操作。同时，在请求失败和响应失败时，我们都可以进行特定的错误处理。

## 取消HTTP请求

在完成搜索相关的功能时，我们经常会需要频繁的发送请求来进行数据查询的情况。通常来说，我们在下一次请求发送时，就需要取消上一次请求。因此，取消请求相关的功能也是一个优点。axios取消请求的示例代码如下：

```javascript
const CancelToken = axios.CancelToken;
const source = CancelToken.source();

axios.get('/user/12345', {
  cancelToken: source.token
}).catch(function(thrown) {
  if (axios.isCancel(thrown)) {
    console.log('Request canceled', thrown.message);
  } else {
    // handle error
  }
});

axios.post('/user/12345', {
  name: 'new name'
}, {
  cancelToken: source.token
})

// cancel the request (the message parameter is optional)
source.cancel('Operation canceled by the user.');
```

通过上面的示例我们可以看到，axios使用的是基于CancelToken的一个撤回提案。不过，目前该提案已经被撤回，具体详情可以见[此处](https://github.com/tc39/proposal-cancelable-promises)。具体的撤回实现方法我们会在后面的章节源码分析的时候进行说明。

# axios的核心模块是如何设计与实现的

通过上面的例子，我相信大家对axios的使用方法都有了一个大致的了解。下面，我们将按照模块来对axios的设计与实现进行分析。下图是我们在这篇博客中将会涉及到的相关的axios的文件，如果读者有兴趣的话，可以通过clone相关代码结合博客进行阅读，这样能够加深对相关模块的理解。

![](https://user-gold-cdn.xitu.io/2018/5/5/1633106961f0e068?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

## HTTP请求模块

作为核心模块，axios发送请求相关的代码位于`core/dispatchReqeust.js`文件中。由于篇幅有限，下面我选取部分重点的源码进行简单的介绍：

```javascript
module.exports = function dispatchRequest(config) {
    throwIfCancellationRequested(config);

    // 其他源码

    // default adapter是一个可以判断当前环境来选择使用Node还是XHR进行请求发送的模块
    var adapter = config.adapter || defaults.adapter; 

    return adapter(config).then(function onAdapterResolution(response) {
        throwIfCancellationRequested(config);

        // 其他源码

        return response;
    }, function onAdapterRejection(reason) {
        if (!isCancel(reason)) {
            throwIfCancellationRequested(config);

            // 其他源码

            return Promise.reject(reason);
        });
};
```

通过上面的代码和示例我们可以知道，`dispatchRequest`方法是通过获取`config.adapter`来得到发送请求的模块的，我们自己也可以通过传入符合规范的adapter函数来替换掉原生的模块（虽然一般不会这么做，不过也算是一个松耦合扩展点）。

在`default.js`文件中，我们能够看到相关的adapter选择逻辑，即根据当前容器中特有的一些属性和构造函数来进行判断。

```javascript
function getDefaultAdapter() {
    var adapter;
    // 只有Node.js才有变量类型为process的类
    if (typeof process !== 'undefined' && Object.prototype.toString.call(process) === '[object process]') {
        // Node.js请求模块
        adapter = require('./adapters/http');
    } else if (typeof XMLHttpRequest !== 'undefined') {
        // 浏览器请求模块
        adapter = require('./adapters/xhr');
    }
    return adapter;
}
```

axios中XHR模块较为简单，为XMLHTTPRequest对象的封装，我们在这里就不过多进行介绍了，有兴趣的同学可以自行阅读，代码位于`adapters/xhr.js`文件中。

## 拦截器模块

了解了`dispatchRequest`实现的HTTP请求发送模块，我们来看下axios是如何处理请求和响应拦截函数的。让我们看下axios中请求的统一入口`request`函数。

```javascript
Axios.prototype.request = function request(config) {

    // 其他代码

    var chain = [dispatchRequest, undefined];
    var promise = Promise.resolve(config);

    this.interceptors.request.forEach(function unshiftRequestInterceptors(interceptor) {
        chain.unshift(interceptor.fulfilled, interceptor.rejected);
    });

    this.interceptors.response.forEach(function pushResponseInterceptors(interceptor) {
        chain.push(interceptor.fulfilled, interceptor.rejected);
    });

    while (chain.length) {
        promise = promise.then(chain.shift(), chain.shift());
    }

    return promise;
};
```

这个函数是axios发送请求的入口，因为函数实现比较长，我就简单说一下相关的设计思路：

1. chain是一个执行队列。这个队列的初始值，是一个带有config参数的Promise。
2. 在chain执行队列中，插入了初始的发送请求的函数`dispatchReqeust`和与之对应的`undefined`。后面需要增加一个`undefined`是因为在Promise中，需要一个success和一个fail的回调函数，这个从代码`promise = promise.then(chain.shift(), chain.shift());`就能够看出来。因此，`dispatchReqeust`和`undefined`我们可以成为一对函数。
3. 在chain执行队列中，发送请求的函数`dispatchReqeust`是处于中间的位置。它的前面是请求拦截器，通过`unshift`方法放入；它的后面是响应拦截器，通过`push`放入。要注意的是，这些函数都是成对的放入，也就是一次放入两个。

通过上面的`request`代码，我们大致知道了拦截器的使用方法。接下来，我们来看下如何取消一个HTTP请求。

## 取消请求模块

取消请求相关的模块在`Cancel/`文件夹中。让我们来看下相关的重点代码。

首先，让我们来看下元数据`Cancel`类。它是用来记录取消状态一个类，具体代码如下：

```javascript
    function Cancel(message) {
      this.message = message;
    }

    Cancel.prototype.toString = function toString() {
      return 'Cancel' + (this.message ? ': ' + this.message : '');
    };

    Cancel.prototype.__CANCEL__ = true;
```

而在CancelToken类中，它通过传递一个Promise的方法来实现了HTTP请求取消，然我们看下具体的代码：

```javascript
function CancelToken(executor) {
    if (typeof executor !== 'function') {
        throw new TypeError('executor must be a function.');
    }

    var resolvePromise;
    this.promise = new Promise(function promiseExecutor(resolve) {
        resolvePromise = resolve;
    });

    var token = this;
    executor(function cancel(message) {
        if (token.reason) {
            // Cancellation has already been requested
            return;
        }

        token.reason = new Cancel(message);
        resolvePromise(token.reason);
    });
}

CancelToken.source = function source() {
    var cancel;
    var token = new CancelToken(function executor(c) {
        cancel = c;
    });
    return {
        token: token,
        cancel: cancel
    };
};
```

而在`adapter/xhr.js`文件中，有与之相对应的取消请求的代码：

```javascript
if (config.cancelToken) {
    // 等待取消
    config.cancelToken.promise.then(function onCanceled(cancel) {
        if (!request) {
            return;
        }

        request.abort();
        reject(cancel);
        // 重置请求
        request = null;
    });
}
```

结合上面的取消HTTP请求的示例和这些代码，我们来简单说下相关的实现逻辑：

1. 在可能需要取消的请求中，我们初始化时调用了source方法，这个方法返回了一个`CancelToken`类的实例A和一个函数cancel。
2. 在source方法返回实例A中，初始化了一个在pending状态的promise。我们将整个实例A传递给axios后，这个promise被用于做取消请求的触发器。
3. 当source方法返回的cancel方法被调用时，实例A中的promise状态由pending变成了fulfilled,立刻触发了then的回调函数，从而触发了axios的取消逻辑——`request.abort()`。

# axios的设计有什么值得借鉴的地方

## 发送请求函数的处理逻辑

在之前的章节中有提到过，axios在处理发送请求的`dispatchRequest`函数时，没有当做一个特殊的函数来对待，而是采用一视同仁的方法，将其放在队列的中间位置，从而保证了队列处理的一致性，提高了代码的可阅读性。

## Adapter的处理逻辑

在adapter的处理逻辑中，axios没有把http和xhr两个模块（一个用于Node.js发送请求，另一个则用于浏览器端发送请求）当成自身的模块直接在`dispatchRequest`中直接饮用，而是通过配置的方法在`default.js`文件中进行默认引入。这样既保证了两个模块间的低耦合性，同时又能够为今后用户需要自定义请求发送模块保留了余地。

## 取消HTTP请求的处理逻辑

在取消HTTP请求的逻辑中，axios巧妙的使用了一个Promise来作为触发器，将resolve函数通过callback中参数的形式传递到了外部。这样既能够保证内部逻辑的连贯性，也能够保证在需要进行取消请求时，不需要直接进行相关类的示例数据改动，最大程度上避免了侵入其他的模块。

# 总结

本文对axios相关的使用方式、设计思路和实现方法进行了详细的介绍。读者能够通过上述文章，了解axios的设计思想，同时能够在axios的代码中，学习到关于模块封装和交互等相关的经验。

由于篇幅原因，本文仅针对axios的核心模块进行了分解和介绍，如果对其他代码有兴趣的同学，可以去[GitHub](https://github.com/axios/axios)进行查看。

如果有任何疑问或者观点，欢迎随时留言讨论。
