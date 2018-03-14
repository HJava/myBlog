# 背景
事件监听在前端的开发过程中是一个很常见的情况。DOM上的事件监听方式，让我们看到了通过事件的方式来进行具体的业务逻辑的处理的便捷。

在具体的一些业务场景中，第三方的自定义事件能够在层级较多，函数调用困难以及需要多个地方响应的时候有着其独特的优势——调用方便，避免多层嵌套，降低组件间耦合性。

这篇文章所提到的EventEmitter3，就是一个典型的第三方事件库，能够让我们通过自定义的实践来实现多个函数与组件间的通信。

# 整体结构图
EventEmitter3的设计较为的简单，具体结构可以看下图所示。

![](https://raw.githubusercontent.com/HJava/blog_picture/master/EventEmitter%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/eventEmitter%E7%BB%93%E6%9E%84.png)

下面我们将按照一般人的正常思路来对这个结构进行介绍。

# 各部分结构与功能
## `EE`

	function EE(fn, context, once) {
	    this.fn = fn;
	    this.context = context;
	    this.once = once || false;
	}
从类`EE`的代码中我们能够很明确的了解到，第一个参数为回调函数，第二个参数为回调函数的上下文，第三个参数是一个`once`的标志位。由于代码简单，在这里就简单介绍下了。

## Prototype属性
### events
该方法用于存储eventEmitter的整个事件名称与回调函数的集合，初始值为undefined。

## Prototype方法
### eventName
- 作用：返回当前已经注册的事件名称的列表
- 参数：无


### listeners
- 作用：返回某一个事件名称的所有监听函数
- 参数：event——事件名称，exists——是否只判断存在与否

### emit
- 作用：触发某个事件
- 参数：event——事件名，a1~a5——参数1~5

### on
- 作用：为某个事件添加一个监听函数
- 参数：event——事件名，fn——回调函数，context——上下文

### once
- 作用：类似`on`,区别在于该函数只会触发一次
- 参数：event——事件名，fn——回调函数，context——上下文


### removeListner
- 作用：移除某个事件的监听函数
- 参数：event——事件名，fn——事件监听函数，context——只移除上下文匹配的事件监听函数，once——只移除类型匹配的事件监听函数

### removeAllListener
- 作用：移除某个时间的所有监听函数
- 参数：event——事件名

# 学习思路
下面我们将从添加监听函数， 事件触发与删除监听函数来进行具体的代码分析，从而了解该库的实现思路。
## 事件对象
具体代码如下所示：

```javascript
 //一个单一的事件监听函数的单元
 //
 // @param {Function} fn Event handler to be called. 回调函数
 // @param {Mixed} context Context for function execution. 函数执行上下文
 // @param {Boolean} [once=false] Only emit once 是否执行一次的标志位
 // @api private 私有API
 
function EE(fn, context, once) {
    this.fn = fn;
    this.context = context;
    this.once = once || false;
}
```
该类为eventEmitter中用于存储事件监听函数的最小类。
## 添加监听函数
`on`函数具体代码如下所示：

```javascript
 // Register a new EventListener for the given event.
 // 注册一个指定的事件的事件监听函数
 //
 // @param {String} event Name of the event. 事件名
 // @param {Function} fn Callback function. 回调函数
 // @param {Mixed} [context=this] The context of the function. 上下文
 // @api public 公有API
 
 EventEmitter.prototype.on = function on(event, fn, context) {
    var listener = new EE(fn, context || this)
        , evt = prefix ? prefix + event : event;

    if (!this._events) this._events = prefix ? {} : Object.create(null);
    if (!this._events[evt]) {
        this._events[evt] = listener;//第一次存储为一个事件监听对象
    } else {
        if (!this._events[evt].fn) {//第三次及以后则直接向对象数组中添加事件监听对象
            this._events[evt].push(listener);
        } else {//第二次将存储的对象与新对象转换为事件监听对象数组
            this._events[evt] = [
                this._events[evt], listener
            ];
        }
    }

    return this;
}
```

当我们向事件E添加函数F时，会调用`on`方法，此时on方法会检查eventEmitter中prototype属性events的E属性。

- 当这个属性为`undefined`时，直接将该函数所在的事件对象赋值给evt属性。
- 当该属性当前值为一个对象且其函数fn不等于函数F时，则会将其转换为一个包含这两个事件对象的事件对象数组。
- 当这个属性已经是一个对象数组时，则直接通过`push`方法向数组中添加对象。

`prefix`是用来判断`Object.create()`方法是否存在，如果存在则直接调用该方法来创建属性，否则通过在属性前添加`~`来避免覆盖原有属性。

`once`的函数实现与`on`函数基本一致，所以在此就不再进行分析。

## 触发监听函数
`emit`函数代码如下所示：

```javascript
// Emit an event to all registered event listeners.
// 触发已经注册的事件监听函数
//
// @param {String} event The name of the event. 事件名
// @returns {Boolean} Indication if we've emitted an event. 如果触发事件成功,则返回true,否则返回false
// @api public 公有API

EventEmitter.prototype.emit = function emit(event, a1, a2, a3, a4, a5) {
    var evt = prefix ? prefix + event : event;

    if (!this._events || !this._events[evt]) return false;

    var listeners = this._events[evt]
        , len = arguments.length
        , args
        , i;

    if ('function' === typeof listeners.fn) {
        if (listeners.once) this.removeListener(event, listeners.fn, undefined, true);

        switch(len) {
            case 1:
                return listeners.fn.call(listeners.context), true;
            case 2:
                return listeners.fn.call(listeners.context, a1), true;
            case 3:
                return listeners.fn.call(listeners.context, a1, a2), true;
            case 4:
                return listeners.fn.call(listeners.context, a1, a2, a3), true;
            case 5:
                return listeners.fn.call(listeners.context, a1, a2, a3, a4), true;
            case 6:
                return listeners.fn.call(listeners.context, a1, a2, a3, a4, a5), true;
        }

        for(i = 1, args = new Array(len - 1); i < len; i++) {
            args[i - 1] = arguments[i];
        }

        listeners.fn.apply(listeners.context, args);
    } else {
        //由于篇幅原因省略E属性为数组时通过循环调用来实现事件触发的过程
    }

    return true;
};
```
当我们触发事件E时，我们只需要调用`emit`方法。该方法会自动检索事件E中所有的事件监听对象，触发所有的事件监听函数，同时移除掉通过`once`添加，只需要触发一次的事件监听函数。

## 移除事件监听函数
`removeListener`函数代码如下：

```javascript
// Remove event listeners.
// 移除事件监听函数
//
// @param {String} event The event we want to remove. 需要被移除的事件名
// @param {Function} fn The listener that we need to find. 需要被移除的事件监听函数
// @param {Mixed} context Only remove listeners matching this context. 只移除匹配该参数指定的上下文的监听函数
// @param {Boolean} once Only remove once listeners. 只移除匹配该参数指定的once属性的监听函数
// @api public 公共API

EventEmitter.prototype.removeListener = function removeListener(event, fn, context, once) {
    var evt = prefix ? prefix + event : event;

    if (!this._events || !this._events[evt]) return this;

    var listeners = this._events[evt]
        , events = [];

    if (fn) {
        if (listeners.fn) {
            if (
                listeners.fn !== fn
                || (once && !listeners.once)
                || (context && listeners.context !== context)
            ) {
                events.push(listeners);
            }
        } else {
            //由于篇幅原因省去便利listeners属性查找函数删除的代码
        }
    }

    //
    // Reset the array, or remove it completely if we have no more listeners.
    //
    if (events.length) {
        this._events[evt] = events.length === 1 ? events[0] : events;
    } else {
        delete this._events[evt];
    }

    return this;
};
```
`removeListener`函数实现较为简单。当我们需要移除事件E的某个函数时，它使用一个`event`属性来保存不需要被移除的事件监听对象，然后便利整个事件监听数组（单个时为对象），并且最后将`event`属性的值赋值给E属性从而覆盖掉原有的属性，达到删除的目的。

## 其他
该库中还有一些其他的函数，由于对整个库的理解不产生太大影响，因此没有在此进行讲解，有需要的可以前往我的[github仓库](https://github.com/HJava/eventemitter3)进行查看。

# 缺点
`eventEmitter`的代码虽然结构清晰，但是仍然存在一些问题。例如`on`和`once`方法的实现中，只有一个属性不同，其余代码都一模一样，其实可以抽出一个特定的函数来进行处理，通过属性来进行区分调用即可。

同时，在同一个函数例如`emit`中，也存在大量的重复代码，可以进行进一步的抽象和整理，使得代码更加简单。

# 总结
`eventEmitter`第三方事件库从实现上来看较为简单，并且结构清晰容易阅读，推荐有兴趣的可以花大约一个小时的时间来学习下。

# 附录
[本人github地址——eventEmitter3](https://github.com/HJava/eventemitter3)
[官方github地址](https://github.com/primus/eventemitter3)
