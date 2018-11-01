# 背景

在上一篇博客[[译]前端基础知识储备——Promise/A+规范](https://cloud.tencent.com/developer/article/1351114)中，我们介绍了Promise/A+规范的具体条目。在本文中，我们来选择了promiz，让大家来看下一个具体的Promise库的内部代码是如何运作的。

[promiz](https://github.com/Zolmeister/promiz/blob/master/promiz.js)是一个体积很小的promise库（官方介绍约为913 bytes (gzip)），作为一个ES2015标准中的Promise的polyfill，实现了诸如`resolve`、`all`和`race`等API。

# 知识储备

我们在这里简单回顾一下Promise/A+的主要关键点，如果需要了解详细内容的同学，可以阅读我的上一篇博客。

- Promise有三个状态，分别为`pending`、`fulfilled`和`rejected`，且只能从`pending`到`fulfilled`或者`rejected`，没有其他的流转方式。
- Promise的返回值是一个新的Promise，原因见上一条。
- 传递给`then`函数的两个回调函数，有且仅有一次机会被执行（即执行了`onfulfilled`就不会执行`onrejected`函数，且只执行一次）。

# 代码实现与分析

## 异步执行器

在介绍Promise之前，我们先介绍一下异步执行器。在Promise中，我们需要一个异步的执行器来异步执行我们的回调函数。在规范中提到，通常情况下，我们可以使用微任务（nextTick）或者宏任务（setTimeout）来实现。但是，如果我们需要兼容Web Worker这种情况的话，我们可能还需要一些更多的方式来处理。具体代码如下：

```javascript
var queueId = 1
var queue = {}
var isRunningTask = false

// 使用postMessage来执行异步函数
if (!global.setImmediate)
    global.addEventListener('message', function (e) {
        if (e.source == global) {
            if (isRunningTask)
                nextTick(queue[e.data])
            else {
                isRunningTask = true
                try {
                    queue[e.data]()
                } catch (e) {}

                delete queue[e.data]
                isRunningTask = false
            }
        }
    })

/**
 * 异步执行方法
 * @param {function} fn 需要执行的回调函数
 */
function nextTick(fn) {
    if (global.setImmediate) setImmediate(fn)
    // 如果在Web Worker中使用以下方法
    else if (global.importScripts) setTimeout(fn)
    else {
        queueId++
        queue[queueId] = fn
        global.postMessage(queueId, '*')
    }
}
```

以上代码比较简单，我们简单说明下：

- 在代码中，promiz使用了`setImmediate`、`setTimeout`和`postMessage`这三个方法来执行异步函数，其中：
  - `setImmedeate`，只有IE实现了该方法，在执行完队列中的代码后立即执行。
  - `PostMessage`，新增的H5中的方法。
  - `setTimeout`，兼容性最佳，可以适用各种场景。

因此，在promiz的这段代码中，有一定的兼容性问题，应该把setTimeout放到最后作为一个兜底策略，否则无法在老浏览器中执行。

## 构造函数

说完了异步函数执行器，我们来看下promise的构造函数。

首先我们来看下内存数据，我们需要存储当前promise的状态、成功的值或者失败的原因、下一个promise的引用和成功与失败的回调函数。因此，我们需要以下变量：

```javascript
// states
// 0: pending
// 1: resolving
// 2: rejecting
// 3: resolved
// 4: rejected
var self = this,
    state = 0, // promise状态
    val = 0, // success callback返回值
    next = [], // 返回的新的promise对象
    fn, er; // then方法中的成功回调函数和失败回调函数
```

在存储完相关数据后，我们来看下构造函数。

```javascript
function Deferred(resolver) {
	...
    self = this;
    try {
        if (typeof resolver == 'function')
            resolver(self['resolve'], self['reject'])
    } catch (e) {
        self['reject'](e)
    }
}
```

构造函数非常简单，除了声明相关的函数，就只有执行传入的callback而已。当然，如果我们不是链式调用的第一个promise，那么我们会没有`resolver`参数，因此不需要在此执行，我们会在`then`函数执行`resolve`方法。

下面我们来看下上面提到的处理函数`resovle`和`reject`。

```javascript
self['resolve'] = function (v) {
    fn = self.fn
    er = self.er
    if (!state) {
        val = v
        state = 1

        nextTick(fire)
    }
    return self
}

self['reject'] = function (v) {
    fn = self.fn
    er = self.er
    if (!state) {
        val = v
        state = 2

        nextTick(fire)

    }
    return self
}

self['then'] = function (_fn, _er) {
    if (!(this._d == 1))
        throw TypeError()

    var d = new Deferred()

    d.fn = _fn
    d.er = _er
    if (state == 3) {
        d.resolve(val)
    }
    else if (state == 4) {
        d.reject(val)
    }
    else {
        next.push(d)
    }

    return d
}
```

在`resolve`和`reject`这两个函数中，都是改变了内部promise的状态，给定了参数值，同时异步触发了fire函数。而`then`方法，则是生成了一个新的`Deferred`对象，并且完成了相关的初始化（执行完then方法我们就会得到这个新生成的`Deferred`对象，也就是一个新的Promise）；当前一个promise到达`resolved`状态时，不需要等待则直接出发resolve方法，`rejected`状态时也一样。那么，让我们来看下fire方法到底是做什么的呢？

```javascript
function fire() {

    // 检测是不是一个thenable对象
    var ref;
    try {
        ref = val && val.then
    } catch (e) {
        val = e
        state = 2
        return fire()
    }

    thennable(ref, function () {
        state = 1
        fire()
    }, function () {
        state = 2
        fire()
    }, function () {
        try {
            if (state == 1 && typeof fn == 'function') {
                val = fn(val)
            }

            else if (state == 2 && typeof er == 'function') {
                val = er(val)
                state = 1
            }
        } catch (e) {
            val = e
            return finish()
        }

        if (val == self) {
            val = TypeError()
            finish()
        } else thennable(ref, function () {
            finish(3)
        }, finish, function () {
            finish(state == 1 && 3)
        })

    })
}
```

从上面的代码来看，fire函数只是判断了ref是不是一个thenable对象，然后调用了thenable函数，传递了3个回调函数。那么这些回调函数到底是做什么用的呢？我们需要来看下thenable函数的实现代码。

```javascript
// ref:指向thenable对象的`then`函数
// cb, ec, cn : successCallback, failureCallback, notThennableCallback
function thennable(ref, cb, ec, cn) {
    // Promises can be rejected with other promises, which should pass through
    if (state == 2) {
        return cn()
    }
    if ((typeof val == 'object' || typeof val == 'function') && typeof ref == 'function') {
        try {

            // cnt变量用来保证成功和失败的回调函数总共只会被执行一次
            var cnt = 0
            ref.call(val, function (v) {
                if (cnt++) return
                val = v
                cb()
            }, function (v) {
                if (cnt++) return
                val = v
                ec()
            })
        } catch (e) {
            val = e
            ec()
        }
    } else {
        cn()
    }
};
```

在thenable函数中，如果判断当前的promise的状态是处于`rejecting`时，会直接执行`cn`，也就是将reject状态传递下去。而如果当ref不是一个thenable对象的`then`函数时（那么此时值为undefined），那么就会直接执行`cn`。

通过fire函数传递的三个callback我们可以看到，`cn`是在promise的状态改变时，针对特定的状态来触发相对应的`onfulfilled`或者`onrejected`回调函数。

只有当`ref`是一个`thenable`时（传递给`resolve`的是一个promise），代码才会进入上面的`try catch`逻辑中。

## Promise执行流程

看完了上面的各部分代码，我相信大家可能对整个执行流程仍然不够熟悉，下面，我们将这些流程拼接起来，通过几个完整的流程来说明下。

## 链式调用第一个Promise

当我们声明一个promise式，我们会传入一个`resolver`。此时，整个`Deferred`对象的`state`是0。如果我们在`resolver`里面调用了`resolve`方法，那么我们的`state`就会变成1，然后出发fire函数注册到thenable函数里面的第三个回调函数，从而将值传递给下一个thenable。当thenable的then函数执行完成（即我们看到的Promise后面跟着的then函数执行完成以后），我们的`state`才会变成3，也就是说上一个Promise才会结束，返回一个新的Promise。

## 链式调用非第一个Promise

如果不是第一个Promise，那么我们就没有`resolver`参数。因此，我们的`resolve`方法并不是通过在`resolver`中进行调用的，而是将回调函数`fn`注册进来，在上一个Promise完成后主动调用执行的。也就是说，我们在上一个Promise执行完then函数并且返回一个新的Promise时，我们这个返回的Promise就已经进入了`resolving`的状态。

## 给`resolve`传递一个Promise

在Promise/A+规范中，如果我们给`resolve`传递一个promise，那么我们的通过`resolve`获取到的值就是传递进去的这个promise返回的值。当然，我们也必须等待作为参数的这个promise处理完成后，才会处理外面的这个promise。

在promiz的代码中，我们如果通过`resolve`接收到一个promise，那么我们在fire函数中就会吧`promise.then`的引用传递给`thenable`函数。在`thenable`函数中，我们会将我们当前promise需要执行的`onfulfilled`和`onrejected`封装成一个函数，传递给作为参数的promise的then函数。因此，当作为参数的promise执行任意结果的回调函数时，就会将参数传递给外层的promise，执行对应的回调函数。

## 全局执行方法

## Promise.all

让我们先看代码。

```javascript
Deferred.all = function (arr) {
    if (!(this._d == 1))
        throw TypeError()

    if (!(arr instanceof Array))
        return Deferred.reject(TypeError())

    var d = new Deferred()

    function done(e, v) {
        if (v)
            return d.resolve(v)

        if (e)
            return d.reject(e)

        var unresolved = arr.reduce(function (cnt, v) {
            if (v && v.then)
                return cnt + 1
            return cnt
        }, 0)

        if (unresolved == 0)
            d.resolve(arr)

        arr.map(function (v, i) {
            if (v && v.then)
                v.then(function (r) {
                    arr[i] = r
                    done()
                    return r
                }, done)
        })
    }

    done()

    return d
}
```

在`Promise.all`中，我们使用了一个计数器来进行统计，在每一个Promise后面都增加一个then函数用于增加计数。当Promise成功时则计数+1。当整个数组中的Promise都已经进入`resolved`状态时，我们才会执行`thenable`的then函数。如果有一个失败的话，则立即进入reject流程。

# 总结

从代码设计层面来看，promiz的代码量较少，阅读也较为简单。但是，在某些细节的设计上，promiz还是体现出了较为巧妙的思路，如在处理作为入参的promise时，能够在这个promise后面动态的添加一个`then`函数，从而获取数据给外面的promise。

如果大家有兴趣，建议自己根据本文的说明阅读一遍[源码](https://github.com/Zolmeister/promiz/blob/master/promiz.js)，配合Promise/A+规范来看下是如何实现每一条规范的。

下一篇博客，我们将为大家从头开始，来实现一个Promise库。