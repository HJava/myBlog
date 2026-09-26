# 概述

自从准备晋级之后，就拖更了很久了，既然晋级弄完了，那么也恢复更新了。

在面试别人的过程中，发现基本上没有人对整个Promise完全了解，因此希望通过这篇文章来帮助大家了解下Promise的全貌。本文的主要内容是Promise/A+规范的译文，主要是能够帮助大家了解下Promise的规范，以及解决在日常运用中遇到的一些问题。

原文地址为：https://promisesaplus.com/#point-1

如果需要测试一个Promise库是否符合Promise/A+规范，可以使用这个：https://github.com/promises-aplus/promises-tests。

# 原文概览

一个开放、健全且可互操作的JavaScript Promise标准——出自实现者，为了实现者。

一个promise代表一个异步操作的最终结果。与promise交互的主要方法是通过它的`then`函数。这个函数注册回调函数来接收promise最终的值（value）或者是没有成功的原因（reason）。

这篇文档详细说明了`then`方法的具体行为，它为所有的符合Promise/A+规范的实现提供了一个基础的交互。因此，这个规范应该被视为非常稳定的。尽管Promise/A+组织可能偶尔通过一些向后兼容的小修改来调整这个文档来适应新发现的边界情况，我们只会在经历过了小心的思考、讨论和测试后增加大的、不向后兼容的修改。

从历史上来看，Promise/A+澄清了更早的Promise/A的行为规范，并且在其基础上进行了扩展，覆盖了一些事实上的行为，同时省略了其中规定不够明确或者有问题的部分。

最终，核心的Promise/A+文档不关心如何去创建、完成（fulfill）或者拒绝（reject）一个Promise，而是聚焦在提供一个可交互的`then`函数。在将来的其他规范中可能会涉及这些没有提及的内容。

# 1. 术语

1.1. "promise"是一个对象或者函数，它拥有一个符合文档中描述行为的`then`方法。

1.2. "thenable"是一个有`then`方法的对象或者函数。

1.3. "value"是一个合法的JavaScript值（包括`undefined`, 一个thenable或者一个promise）。

1.4. "exception"是一个使用在`throw`语句中的抛出来的值。

1.5. "reason"是一个用来表示promise拒绝的原因的值。

# 2. 要求

## 2.1 promise状态

一个promise必须处于以下三种状态：pending、fulfilled或者rejected。

2.1.1. 当处于pending状态时，promise：

2.1.1.1. 可能会转换成fulfilled状态或者rejected状态。

2.1.2. 当处于fulfilled状态时，promise：

2.1.2.1. 禁止转换成其他状态。

2.1.2.2. 必须有一个无法更改的值。

2.1.3. 当处于rejected状态时，promise：

2.1.3.1. 禁止转换成其他状态。

2.1.3.2. 必须有一个无法更改的原因。

在这里，无法更改意味着引用不可变（即`===`），但是不代表深层次的不可变。

## 2.2 `then`方法

promise必须包含一个`then`方法来访问它当前或者最终的值或者原因。

Promise的`then`方法接收两个参数：

```javascript
promise.then(onFulfilled, onRejected)
```

2.2.1. `onFulfilled`和`onRejected`都是可选的参数：

2.2.1.1. 如果`onFulfilled`不是一个函数，那么它必须被忽略掉。

2.2.1.2. 如果`onRejected`不是一个函数，那么它必须被忽略掉。

2.2.2. 如果`onFulfilled`是一个函数：

2.2.2.1. 它必须在`promise`到fulfilled状态后触发，`promise`的值是它的第一个参数。

2.2.2.2. 它在一个`promise`到fulfilled状态之前禁止被触发。

2.2.2.3. 它禁止被触发多次。

2.2.3. 如果`onRejected`是一个函数：

2.2.3.1. 它必须在`promise`到rejected状态后触发，`promise`的原因是它的第一个参数。

2.2.3.2. 它在一个`promise`到rejected之前禁止被触发。

2.2.3.3. 它禁止被触发多次。

2.2.4. `onFulfilled`或者`onRejected`只有在执行上下文堆栈只有平台代码时才能被触发。

2.2.5. `onFulfilled`和`onRejected`必须作为函数被调用（即没有`this`值）。

2.2.6. `then`方法可能在相同的promise中被调用多次。

2.2.6.1. 如果`promise`到了fulfilled状态，那么所有的`onFulfilled`回调函数都必须按照他们原有的顺序进行调用执行。

2.2.6.2. 如果`promise`到了rejected状态，那么所有的`onRejected`回调函数都必须按照他们原有的顺序进行调用执行。

2.2.7. `then`方法必须返回一个promise：

```javascript
promise2 = promise1.then(onFulfilled, onRejected);
```

2.2.7.1. 如果`onFulfilled`或者`onRejected`方法返回一个值`x`，那么执行promise的解析过程`[[Resolve]](promise2, x)`。

2.2.7.2. 如果`onFulfilled`或者`onRejected`方法抛出一个异常`e`，`promise2`必须使用`e`作为原因拒绝掉（rejected）。

2.2.7.3. 如果`onFulfilled`不是一个函数并且`promise1`到了fulfilled状态，那么`promise2`必须使用与`promise1`相同的值转换到fulfilled状态。

2.2.7.4. 如果`onRejected`不是一个函数并且`promise1`到了rejected状态，那么`promise2`必须使用与`promise1`相同的原因转换到rejected状态。

## 2.3. promise解析函数

promise解析函数是一个输入一个promise和一个值的抽象的操作，我们表示为`[[Resolve]](promise, x)`。如果`x`是一个thenable对象，在假定`x`的行为至少有点像一个promise的情况下，它会尝试让`promise`转换到`x`的状态。否则，它会用`x`的值完成`promise`的状态。

这种thenable对象的方式允许promise实现交互，只要他们暴露一个符合Promise/A+规范的`then`函数。它还允许Promise/A+的实现“同化”那些拥有合理`then`方法的不符合规范的实现。

运行`[[Resolve]](promise, x)`，需要遵循以下步骤：

2.3.1. 如果`promise`和`x`指向同一个对象，那么用`TypeError`作为原因拒绝promise。

2.3.2. 如果`x`是一个promise，采用它的状态：

2.3.2.1. 如果`x`是pending状态，`promise`保留pending状态直到`x`变成fulfilled状态或者rejected状态。

2.3.2.2. 如果`x`是fulfilled状态，那么用同样的值将整个`promise`完成。

2.3.2.3. 如果`x`是rejected状态，那么用同样的原因拒绝`promise`。

2.3.3. 否则，如果`x`是一个对象或者函数，

2.3.3.1. 让`then`变成`x.then`。

2.3.3.2. 如果在获取`x.then`这个属性的结果时抛出一个异常`e`，把`e`作为原因拒绝`promise`。

2.3.3.3. 如果`then`是一个函数，那么用`x`作为`this`来调用它，第一个参数是`resolvePromise`，第二个参数是`rejectPromise`：

2.3.3.3.1. 如果`resolvePromise`被值`y`调用，那么运行`[[Resolve]](promise, y)`。

2.3.3.3.2. 如果`rejectPromise`被原因`r`触发，那么用`r`来拒绝`promise`。

2.3.3.3.3. 如果`resolvePromise`和`rejectPromise`都被调用，或者同一个函数被多次调用，那么第一次调用生效，其他之后的任何调用都忽略掉。

2.3.3.3.4. 如果在调用`then`方法时抛出了一个异常`e`，

2.3.3.3.4.1. 如果`resolvePromise`或者`rejectPromise`已经被调用了那么就忽略掉它。

2.3.3.3.4.2. 否则，使用`e`作为原因拒绝`promise`。

2.3.3.4. 如果`then`不是一个函数，那么用`x`完成`promise`。

2.3.4. 如果`x`不是一个对象或者函数，那么用`x`完成`promise`。

如果一个promise是通过在环形的thenable链中的一个thenable来完成的，如`[[Resolve]](promise, thenable)`的递归特性最终导致`[[Resolve]](promise, thenable)`被再次调用，遵循上述的规则会导致无穷递归。对这种递归情况的检测并且使用`TypeError`作为原因进行拒绝，我们鼓励实现，但不要求。

# 3. 注意事项

3.1. 在这里"平台代码"（platform code）意味着引擎，环境和promise实现代码。在实践中，这个要求确保了`onFulfilled`和`onRejected`是异步执行的，并且是在调用`then`的那一轮事件循环之后、在一个新的堆栈中执行。这可以通过一个宏任务（macro-task）机制例如`setTimeout`或者`setImmediate`来实现，也可以通过一个微任务（micro-task）例如`MutationObserver`或者`process.nextTick`来实现。由于promise的实现被视为平台代码，因此它自己可能会带一个任务执行队列或者“蹦床”来调用这些处理函数。

3.2. 在严格模式下，`this`在这些回调函数里面将会是`undefined`。在松散模式下，它代表的是全局对象。

3.3. 如果实现满足所有要求的话，可以允许`promise2 === promise1`。每一个实现都应该表明是否支持`promise2 === promise1`，如果支持则是需要在什么条件下。

3.4. 通常来说，只有当`x`来自当前的实现时，我们才能知道它是一个真正的promise。这一条允许使用特定于实现的方式来采用已知符合规范的promise的状态。

3.5. 在程序中，首先存储`x.then`的引用，其次测试这个引用，然后再调用这个引用，避免多次访问`x.then`属性。这样的预防措施对于确保那些会在两次访问之间可能变化的属性值获取到一致的结果非常重要。

3.6. 实现中不应该对thenable调用链的深度设置任意的限制，并假定超出该限制的递归就是无穷递归。只有真正的循环才应该导致`TypeError`；如果遇到的是一条由不同thenable组成的无穷长的链，那么一直递归下去才是正确的行为。

# 总结

本文主要通过英文翻译为中文的Promise/A+规范，让大家了解了整个规范的全部内容。我会在下一篇博客中给大家带来如何实现一个完全符合Promise/A+规范的Promise。