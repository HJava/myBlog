# 概述

从上次更新Promise/A+规范后，已经很久没有更新博客了。之前由于业务需要，完成了一个TypeScript语言的Promise库。这次我们来和大家一步一步介绍下，我们如何实现一个符合Promise/A+规范的Promise库。

如果对Promise/A+规范还不太了解的同学，建议先看看上一篇博客——[ \[译]前端基础知识储备——Promise/A+规范 ][1]。

# 实现流程

首先，我们来看下，在我实现的这一个Promise中，代码由下面这几部分组成：

- 全局异步函数执行器
- 常量与属性
- 类方法
- 类静态方法

通过上面这四个部分，我们就能够得到一个完整的Promise。这四个部分互相有关联，接下来我们一个一个模块来看。

## 全局异步函数执行器

在之前的[Promiz的源码分析][2]的博客中我有提到过，我们如何来实现一个异步函数执行器。通过JavaScript的执行原理我们可以知道，如果要实现异步执行相关函数的话，我们可以选择使用宏任务和微任务，这一点在Promise/A+的规范中也有提及。因此，下面我们提供了一个用宏任务来实现异步函数执行器的代码供大家参考。

```ts
let index = 0;

if (global.postMessage) {
    global.addEventListener('message', (e) => {
        if (e.source === global) {
            let id = e.data;
            if (isRunningTask) {
                nextTick(functionStorage[id]);
            } else {
                isRunningTask = true;

                try {
                    functionStorage[id]();
                } catch (e) {

                }
                isRunningTask = false;
            }

            delete functionStorage[id];
            functionStorage[id] = void 0;
        }
    });
}

function nextTick(func) {
    if (global.setImmediate) {
        global.setImmediate(func);
    } else if (global.postMessage) {
        functionStorage[++index] = func;
        global.postMessage(index, '*')
    } else {
        setTimeout(func);
    }
}
```

通过上面的代码我们可以看到，我们一共使用了`setImmediate`、`postMessage`、`setTimeout`这三个添加宏任务的方法来进行一步函数执行。

## 常量与属性

说完了如何进行异步函数的执行，我们来看下相关的常量与属性。在实现Promise之前，我们需要定义一些常量和类属性，用于后面存储数据。让我们一个一个来看下。

### 常量

首先，Promise共有5个状态，我们需要用常量来进行定义，具体如下：

```ts
enum State {
    pending = 0,
    resolving = 1,
    rejecting = 2,
    resolved = 3,
    rejected = 4
};
```

这五个常量分别对应Promise中的5个状态，相信大家能够从名字中理解，我们就不多讲了。

### 属性

在Promise中，我们需要一些属性来存储数据状态和后续的Promise引用，具体如下：

```ts
class Promise {
	private _value;
	private _reason;
	private _next = [];
	public state: State = 0;
	public fn;
	public er;
}
```

我们对属性进行逐一说明：

- `_value`，表示在resolved状态时，用来存储当前的值。
- `_reason`，表示在rejected状态时，用来存储当前的原因。
- `_next`，表示当前Promise后面跟着`then`函数的引用。
- `fn`，表示当前Promise中的`then`方法的第一个回调函数。
- `er`，表示当前Promise中的`then`方法的的第二个回调函数（即`catch`的第一个参数，下面看`catch`实现方法就能理解）。

## 类方法

看完了常量与类的属性，我们来看下类的静态方法。

### Constructor

首先，如果我们要实现一个Promise，我们需要一个构造函数来初始化最初的Promise。具体代码如下：

```ts
class Promise {
    constructor(resolver?) {
        if (typeof resolver !== 'function' && resolver !== undefined) {
            throw TypeError()
        }


        if (typeof this !== 'object') {
            throw TypeError()
        }

        try {
            if (typeof resolver === 'function') {
                resolver(this.resolve.bind(this), this.reject.bind(this));
            }
        } catch (e) {
            this.reject(e);
        }
    }
}
```

从Promise/A+的规范来看，我们可以知道，如果`resolver`存在并且不是一个function的话，那么我们就应该抛出一个错误；否则，我们应该将`resolve`和`reject`方法传给`resolver`作为参数。

### resolve && reject

那么，`resolve`和`reject`方法又是做什么的呢？这两个方法主要是用来让当前的这个Promise转换状态的，即从`pending`状态转换为`resolving`或者`rejecting`状态。下面让我们来具体看下代码：

```ts
class Promise {
	resolve(value) {
        if (this.state === State.pending) {
            this._value = value;
            this.state = State.resolving;

            nextTick(this._handleNextTick.bind(this));
        }

        return this;
    }

    reject(reason) {
        if (this.state === State.pending) {
            this._reason = reason;
            this.state = State.rejecting;
            this._value = void 0;

            nextTick(this._handleNextTick.bind(this));
        }

        return this;
    }
}
```

从上面的代码中我们可以看到，当`resolve`或者`reject`方法被触发时，我们都改变了当前这个Proimse的状态，并且异步调用执行了`_handleNextTick`方法。状态的改变标志着当前的Promise已经从`pending`状态改变成了`resolving`或者`rejecting`状态，而相应`_value`和`_reson`也表示上一个Promise传递给下一个Promise的数据。

那么，这个`_handleNextTick`方法又是做什么的呢？其实，这个方法的作用很简单，就是用来处理当前这个Promise后面跟着的`then`函数传递进来的回调函数`fn`和`er`。

### then && catch

在了解`_handleNextTick`之前，我们们先看下`then`函数和`catch`函数的实现。

```ts
class Promise {
	public then(fn, er?) {
        let promise = new Promise();
        promise.fn = fn;
        promise.er = er;

        if (this.state === State.resolved) {
            promise.resolve(this._value);
        } else if (this.state === State.rejected) {
            promise.reject(this._reason);
        } else {
            this._next.push(promise);
        }

        return promise;
    }

    public catch(er) {
        return this.then(null, er);
    }
}
```

因为`catch`函数调用就是一个`then`函数的别名，我们下面就只讨论`then`函数。

在`then`函数执行时，我们会创建一个新的Promise，然后将传入的两个回调函数用新的Promise的属性保存下来。然后，先判断当前的Promise的状态，如果已经是`resolved`或者`rejected`状态时，我们立即调用新的Promise中`resolve`或者`reject`方法，让下将当前Promise的`_value`或者`_reason`传递给下一个Promise，并且触发下一个Promise的状态改变。如果当前Promise的状态仍然为`pending`时，那么就将这个新生成的Promise保存下来，等当前这个Promise的状态改变后，再触发新的Promise变化。最后，我们返回了这个Promise的实例。

### handleNextTick

看完了`then`函数，我们就可以来看下我们提到过的`handleNextTick`函数。

```ts
class Promise {
	private _handleNextTick() {
		try {
		    if (this.state === State.resolving && typeof this.fn === 'function') {
		        this._value = this.fn.call(getThis(), this._value);
		    } else if (this.state === State.rejecting && typeof this.er === 'function') {
		        this._value = this.er.call(getThis(), this._reason);
		        this.state = 1;
		    }
		} catch (e) {
		    this.state = State.rejecting;
		    this._reason = e;
		    this._value = void 0;
		    this._finishThisTypeScriptPromise();
		}
		
		// if promise === x, use TypeError to reject promise
		// 如果promise和x指向同一个对象，那么用TypeError作为原因拒绝promise
		if (this._value === this) {
		    this.state = State.rejecting;
		    this._reason = new TypeError();
		    this._value = void 0;
		}
		
		this._finishThisTypeScriptPromise();
	}
}
```

我们先来看一个简单版的`_handleNextTick`函数，这样能够帮助我们快速理解Promise主流程。

异步触发了`_handleNextTick`函数后，我们会判断当前用户处于的状态，如果当前Promise是`resolving`状态，我们就会调用`fn`函数，即我们在`then`函数调用时给新的Promise设置的那个`fn`函数；而如过当前Promise是`rejecting`状态，我们就会调用`er`函数。

上面提到的`getThis`方法是用来获取特定的this值，具体的规范要求我们将在稍后再进行介绍。

通过执行这两个同步的`fn`或`er`函数，我们能够得到当前Promise执行完传入回调后的值。在这里需要说明的是：我们在执行`fn`或者`er`函数之前，我们在`_value`和`_reason`中存放的值，是上一个Promise传递下来的值。只有当执行完了`fn`或者`er`函数后，`_value`和`_reason`中存放的值才是我们需要传递给下一个Promise的值。

大家到这里可能会奇怪，我们的`this`指向没有发生变化，但是为什么我们的`this`指向的是那个新的Promise，而不是原来的那个Promise呢？

我们可以从另外一个角度来看待这个问题：我们当前的这个Promise是不是由上一个Promise所产生的呢？如果是这种情况的话，我们就可以理解，在上一个Promise产生当前Promise的时候，就设置了`fn`和`er`两个函数。

大家可能又会问，那么我们第一个Promise的`fn`和`er`这两个参数是怎么来的呢？

那么我们就需要仔细看下上面这个逻辑了。下面我们只讨论第一个Promise处于`pending`的情况，其余的情况与这种情形基本相同。第一个Promise因为没有上一个Promise去设置`fn`和`er`两个参数，因此这两个参数的值就是`undefined`。所以在上面的逻辑中，我们已经排除了这种情况，直接进入了`_finishThisTypeScriptPromise`函数中。

我们在这里需要特别说明下的是，有些人会认为我们在调用`then`函数传入的两个回调函数`fn`和`er`时，当前Promise就结束了，其实并不是这样，我们是得到了`fn`或者`er`两个函数的返回值，再将值传递给下一个Promise时，上一个Promise才会结束。关于这个逻辑我们可以看下`_finishThisTypeScriptPromise`函数。

### finishThisTypeScriptPromise

`_finishThisTypeScriptPromise`函数的代码如下：

```ts
class Promise {
	private _finishThisTypeScriptPromise() {
        if (this.state === State.resolving) {
            this.state = State.resolved;

            this._next.map((nextTypeScriptPromise) => {
                nextTypeScriptPromise.resolve(this._value);
            });
        } else {
            this.state = State.rejected;

            this._next.map((nextTypeScriptPromise) => {
                nextTypeScriptPromise.reject(this._reason);
            });
        }
    }
}
```

从`_finishThisTypeScriptPromise`函数中我们可以看到，我们在得到了需要传递给下一个Promise的`_value`或者`_reason`后，利用`map`方法逐个调用我们保存的新生成的Promise实例，调用它的`resolve`方法，因此我们又触发了这个Promise的状态从`pending`转变为`resolving`或者`rejecting`。

到这里我们就已经完全了解了一个Promise从最开始的创建，到最后结束的整个生命周期。下面我们来看下在Promise/A+规范中提到的一些分支逻辑的处理情况。

### 上一个Promise传递的value是Thenable实例

首先，让我们来了解下什么是Thenable实例。Thenable实例指的是属性中有`then`函数的对象。Promise就是的一种特殊的Thenable对象。

下面，为了方便讲解，我们将用Promise来代替Thenable进行讲解，其他的Thenable类大家可以参考类似思路进行分析。

如果我们在传递给我们的`_value`中是一个Promise实例，那么我们必须在等待传入的Promise状态转换到`resolved`之后，当前的Promise才能够继续往下执行，即我们从传入的Promise中得到了一个非Thenable返回值时，我们才能用这个值来调用属性中的`fn`或者`er`方法。

那么，我们要怎么样才能获取到传入的这个Promise的返回值呢？在Promise中其实用到了一个非常巧妙的方法：因为传入的Promise中有一个`then`函数（Thenable定义），因此我们就调用`then`函数，在第一个回调函数`fn`中传入获取`_value`，触发当前的Promise继续执行。如果是触发了第二个回调函数`er`，那么就用在`er`中得到的`_reason`来拒绝掉当前的Promise。具体判断逻辑如下：

```ts
class Promise {
	private _handleNextTick() {
        let ref;
        let count = 0;

        try {
            // 判断传入的this._value是否为一个thanable
            // check if this._value a thenable
            ref = this._value && this._value.then;
        } catch (e) {
            this.state = State.rejecting;
            this._reason = e;
            this._value = void 0;

            return this._handleNextTick();
        }

        if (this.state !== State.rejecting && (typeof this._value === 'object' || typeof this._value === 'function') && typeof ref === 'function') {
            // add a then function to get the status of the promise
            // 在原有TypeScriptPromise后增加一个then函数用来判断原有promise的状态

            try {
                ref.call(this._value, (value) => {
                    if (count++) {
                        return;
                    }

                    this._value = value;
                    this.state = State.resolving;
                    this._handleNextTick();
                }, (reason) => {
                    if (count++) {
                        return;
                    }

                    this._reason = reason;
                    this.state = State.rejecting;
                    this._value = void 0;
                    this._handleNextTick();
                });
            } catch (e) {
                this.state = State.rejecting;
                this._reason = e;
                this._value = void 0;
                this._handleNextTick();
            }
        } else {
            try {
                if (this.state === State.resolving && typeof this.fn === 'function') {
                    this._value = this.fn.call(getThis(), this._value);
                } else if (this.state === State.rejecting && typeof this.er === 'function') {
                    this._value = this.er.call(getThis(), this._reason);
                    this.state = 1;
                }
            } catch (e) {
                this.state = State.rejecting;
                this._reason = e;
                this._value = void 0;
                this._finishThisTypeScriptPromise();
            }

            this._finishThisTypeScriptPromise();
        }
    }
}
```

### promise === value

在Promise/A+规范中，如果返回的`_value`值等于用户自身时，需要用TypeError错误拒绝掉当前的Promise。因此我们需要在`_handleNextTick`中加入以下判断代码：

```ts
class Promise {
	    private _handleNextTick() {
        let ref;
        let count = 0;

        try {
            // 判断传入的this._value是否为一个thanable
            // check if this._value a thenable
            ref = this._value && this._value.then;
        } catch (e) {
            this.state = State.rejecting;
            this._reason = e;
            this._value = void 0;

            return this._handleNextTick();
        }

        if (this.state !== State.rejecting && (typeof this._value === 'object' || typeof this._value === 'function') && typeof ref === 'function') {
            // add a then function to get the status of the promise
            // 在原有TypeScriptPromise后增加一个then函数用来判断原有promise的状态
			
			...

        } else {
            try {
                if (this.state === State.resolving && typeof this.fn === 'function') {
                    this._value = this.fn.call(getThis(), this._value);
                } else if (this.state === State.rejecting && typeof this.er === 'function') {
                    this._value = this.er.call(getThis(), this._reason);
                    this.state = 1;
                }
            } catch (e) {
                this.state = State.rejecting;
                this._reason = e;
                this._value = void 0;
                this._finishThisTypeScriptPromise();
            }

            // if promise === x, use TypeError to reject promise
            // 如果promise和x指向同一个对象，那么用TypeError作为原因拒绝promise
            if (this._value === this) {
                this.state = State.rejecting;
                this._reason = new TypeError();
                this._value = void 0;
            }

            this._finishThisTypeScriptPromise();
        }
    }
}
```

### getThis

在Promise/A+规范中规定：我们在调用`fn`和`er`两个回调函数时，this的指向有限制。在严格模式下，this的值应该为`undefined`；在宽松模式下时，this的值应该为`global`。

因此，我们还需要提供一个`getThis`函数用于处理上述情况。具体代码如下：

```ts
class Promise {
	...
}

function getThis() {
	return this;
}
```

## 类静态方法

我们通过上面说到的类方法和一些特定分支的逻辑处理，我们就已经实现了一个符合基本功能的Promise类。那么，下面我们来看下ES6中提供的一些标准API我们如何来进行实现。具体API如下：

- resolve
- reject
- all
- race

下面我们就一个一个方法来看下。

### resolve && reject

首先我们来看下最简单的`resolve`和`reject`方法。

```ts
class Promise {
    public static resolve(value?) {
        if (TypeScriptPromise._d !== 1) {
            throw TypeError();
        }

        if (value instanceof TypeScriptPromise) {
            return value;
        }

        return new TypeScriptPromise((resolve) => {
            resolve(value);
        });
    }

    public static reject(value?) {
        if (TypeScriptPromise._d !== 1) {
            throw TypeError();
        }

        return new TypeScriptPromise((resolve, reject) => {
            reject(value);
        });
    }
}
```

通过上面代码我们可以看到，`resolve`和`reject`方法基本上就是直接使用了内部的`constructor`方法进行Promise构建。

### all

```ts
class Promise {
	public static all(arr) {
        if (TypeScriptPromise._d !== 1) {
            throw TypeError();
        }

        if (!(arr instanceof Array)) {
            return TypeScriptPromise.reject(new TypeError());
        }

        let promise = new TypeScriptPromise();

        function done() {
            // 统计还有多少未完成的TypeScriptPromise
            // count the unresolved promise
            let unresolvedNumber = arr.filter((element) => {
                return element && element.then;
            }).length;

            if (!unresolvedNumber) {
                promise.resolve(arr);
            }

            arr.map((element, index) => {
                if (element && element.then) {
                    element.then((value) => {
                        arr[index] = value;
                        done();
                        return value;
                    });
                }
            });
        }

        done();

        return promise;
    }
}
```

下面我们根据上面的代码来简单说下`all`函数的基本思路。

首先我们需要先创建一个新的Promise用于返回，保证后面用户调用`then`函数进行后续逻辑处理时可以设置新Promise的`fn`和`er`这两个回调函数。

然后，我们怎么获取上面Promise数组中每一个Promise的值呢？方法很简单，我们在前面就已经介绍过：我们调用了每一个Promise的`then`函数用来获取当前这个Promise的值。并且，在每个Promise完成时，我们都检查下是否所有的Promise都已经完成，如果已经完成，则触发新Promise的状态从`pending`转换为`resolving`或者`rejecting`。

### race

```ts
class Promise {
	public static race(arr) {
        if (TypeScriptPromise._d !== 1) {
            throw TypeError();
        }

        if (!(arr instanceof Array)) {
            return TypeScriptPromise.reject(new TypeError());
        }

        let promise = new TypeScriptPromise();

        function done(value?) {
            if (value) {
                promise.resolve(value);
            }

            let unresolvedNumber = arr.filter((element) => {
                return element && element.then;
            }).length;

            if (!unresolvedNumber) {
                promise.resolve(arr);
            }

            arr.map((element, index) => {
                if (element && element.then) {
                    element.then((value) => {
                        arr[index] = value;
                        done(value);
                        return value;
                    });
                }
            });
        }

        done();

        return promise;
    }
}
```

`race`的思路与`all`基本一致。只是我们在处理函数上不同。当我们只要检测到数组中的Promise有一个已经转换到了`resolve`或者`rejected`状态（通过没有`then`函数来进行判断）时，就会立即出发新创建的Promise示例的状态从`pending`转换为`resolving`或者`rejecting`。

# 总结

我们对Promise的异步函数执行器、常量与属性、类方法、类静态方法进行了逐一介绍，让大家对整个Promise的构造和声明周期有了一个深度的理解和认知。在整个开发中需要注意到的一些关键点和细节，我在上面也一一说明了。大家只需要按照这个思路，对照[Promise/A+规范][3]就能够完成一个符合规范的Promise库。

最后，给大家提供一个[Promise/A+测试工具][4]，大家实现了自己的Promise后，可以使用这个工具来测试是否完全符合整个Promise/A+规范。当然，大家如果想使用我的现成代码，也欢迎大家使用我的代码[Github/typescript-proimse][5]。

[1]:	https://juejin.im/post/5b9ce8fe6fb9a05cf3711c98 "[译]前端基础知识储备——Promise/A+规范"
[2]:	https://juejin.im/post/5bdaf5d95188257fa660c0d1
[3]:	https://juejin.im/post/5b9ce8fe6fb9a05cf3711c98
[4]:	https://github.com/promises-aplus/promises-tests
[5]:	https://github.com/HJava/typescript-promise