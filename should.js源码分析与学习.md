# 背景
为了研究与学习某些测试框架的工作原理，同时也为了完成培训中实现一个简单的测试框架的原因，我对should.js的代码进行了学习与分析，现在与大家来进行交流下。

# 目录
- ext
- assertion.js
- assertion-error.js
- config.js
- should.js
- util.js

其中`ext`为文件夹，其余为js文件。

# 结构
其中`should.js`为整个项目入口，`asssertion.js`为should.js中的类，负责对测试信息进行记录。`assertion-error.js`为should.js定义了一个错误类，负责存储错误信息。`config.js`中存储了一些should.js中的一些配置信息。`util.js`中则定义了一些项目中常用的工具函数。
## `should.js`
```javascript
var should = function should(obj) {
	return (new should.Assertion(obj));
};

should.AssertionError = require('./assertion-error');
should.Assertion = require('./assertion');

should.format = util.format;
should.type = require('should-type');
should.util = util;
should.config = require('./config');

exports = module.exports = should;
```

`should.js`入口文件初始化了一个类，并将所有文件中其他的模块进行引入。同时将自己export出去，让自己能够被require到。

```javascript
should.extend = function (propertyName, proto) {
	propertyName = propertyName || 'should';
	proto = proto || Object.prototype;

var prevDescriptor = Object.getOwnPropertyDescriptor(proto, propertyName);

Object.defineProperty(proto, propertyName, {
    set: function () {
    },
    get: function () {
        return should(util.isWrapperType(this) ? this.valueOf() : this);
    },
    configurable: true
});

return {
	name: propertyName, descriptor: prevDescriptor, proto: proto};
};
```

`should.js`自身定义了一个extend方法，用于兼容should.js的另一种调用方式，即`should(obj)`的方式等于`should.js`的常规调用方式`obj.should`，从而兼容另一种写法。
​	
```javascript
should
    .use(require('./ext/assert'))
    .use(require('./ext/chain'))
    .use(require('./ext/bool'))
    .use(require('./ext/number'))
    .use(require('./ext/eql'))
    .use(require('./ext/type'))
    .use(require('./ext/string'))
    .use(require('./ext/property'))
    .use(require('./ext/error'))
    .use(require('./ext/match'))
    .use(require('./ext/contain'));
```

`should.js`中还定义了use方法，从而让我们能够自己编写一些类型判断例如isNumber等函数导入到项目中，从而方便进行测试。项目目录中的`ext`文件夹就是编写的一些简单的should.js的扩展。后面将在介绍扩展时对两者的工作原理以及使用方法进行介绍。
## `assertion.js`
```javascript
function Assertion(obj) {
    this.obj = obj;

    /**
     * any标志位
     * @type {boolean}
     */
    this.anyOne = false;

    /**
     * not标志位
     * @type {boolean}
     */
    this.negate = false;

    this.params = {actual: obj};
}
```

`assertion.js`中定义了一个Assertion类，其中any为should.js中的`any`方法的标志位，而not则为其`not`方法的标志位。

```javascript
Assertion.add = function(name, func) {
    var prop = {enumerable: true, configurable: true};

    prop.value = function() {
        var context = new Assertion(this.obj, this, name);
        context.anyOne = this.anyOne;

        try {
            func.apply(context, arguments);
        } catch(e) {
            //check for fail
            if(e instanceof AssertionError) {
                //negative fail
                if(this.negate) {
                    this.obj = context.obj;
                    this.negate = false;
                    return this;
                }

                if(context !== e.assertion) {
                    context.params.previous = e;
                }

                //positive fail
                context.negate = false;
                context.fail();
            }
            // throw if it is another exception
            throw e;
        }

        //negative pass
        if(this.negate) {
            context.negate = true;//because .fail will set negate
            context.params.details = 'false negative fail';
            context.fail();
        }

        //positive pass
        if(!this.params.operator) this.params = context.params;//shortcut
        this.obj = context.obj;
        this.negate = false;
        return this;
    };

    Object.defineProperty(Assertion.prototype, name, prop);
};
```

`assertion.js`中的add方法在Assertion的原型链中添加自定义命名的方法，从而让我们能够打包一些判断的方法来进行调用，不需要重复进行代码的编写。该方法具体的使用方式我们在后面对扩展进行讲解时将会提到。
​	
```javascript
Assertion.addChain = function(name, onCall) {
    onCall = onCall || function() {
        };
    Object.defineProperty(Assertion.prototype, name, {
        get: function() {
            onCall();
            return this;
        },
        enumerable: true
    });
};
```

`addChain`方法添加属性到原型链中，该属性在调用方法后返回调用者本身。该方法在`should.js`的链式调用中起着重要的作用。

同时，Assertion类还支持别名功能，`alias`方法使用Object对象的`getOwnPropertyDescriptor`方法来对属性是否存在进行判断，并调用`defineProperty`进行赋值。

`Assertion`类在原型链中定义了`assert`方法，用来对各级限制条件进行判断。`assert`方法与普通方法不同，它并未采用参数来进行一些参数的传递，而是通过`assert`方法所在的`Assertion`对象的`params`属性来进行参数的传递。因为在`Assertion`对象中存储了相关的信息，使用这个方法来进行参数传递方便在各级中`assert`函数的调用方便。具体使用方法我们将在扩展的分析时提到。

```javascript
assert: function(expr) {
    if(expr) return this;

    var params = this.params;

    if('obj' in params && !('actual' in params)) {
        params.actual = params.obj;
    } else if(!('obj' in params) && !('actual' in params)) {
        params.actual = this.obj;
    }

    params.stackStartFunction = params.stackStartFunction || this.assert;
    params.negate = this.negate;

    params.assertion = this;

    throw new AssertionError(params);
}
```

`Assertion`类也定义了一个`fail`方法能够让用户直接调用从而抛出一个Assertion的Error。

```javascript
fail: function() {
    return this.assert(false);
}
```

## `assertion-error.js`
在此文件中，定义了assertion中抛出来的错误，同时在其中定义了一些信息存储的函数例如`message`和`detail`等，能够让错误在被捕获的时候带上一些特定的信息从而方便进行判断与处理。由于实现较为简单，因此在此就不贴出代码，需要了解的人可以自己去查阅should.js的源码。

## `ext/bool.js`
下面简单介绍一个`Assertion`的扩展的工作方式。让我们能够对should.js的工作原理有一个更加深刻的理解。

```javascript
module.exports = function(should, Assertion) {
    /**
     * 判断是否为true
     */
    Assertion.add('true', function() {
        this.is.exactly(true);
    });
    /**
     * 别名为True
     */
    Assertion.alias('true', 'True');

    /**
     * 判断是否为false
     */
    Assertion.add('false', function() {
        this.is.exactly(false);
    });
    /**
     * 别名False
     */
    Assertion.alias('false', 'False');

    /**
     * 通过对象检查来判断对象是否为空
     */
    Assertion.add('ok', function() {
        this.params = {operator: 'to be truthy'};

        this.assert(this.obj);
    });
};

//should.js
should.use = function (f) {
    f(should, should.Assertion);
    return this;
};

//use
'1'.should.be.true();
```


通过上面的扩展模块代码以及`should.js`文件中的`use`函数，我们可以发现，`use`函数向扩展模块传入了`should`方法和`Assertion`构造函数。在`bool.js`这个扩展模块中，它通过调用`Assertion`对象上的add函数来添加新的判断方式，并且通过`params`参数来告诉`Assertion`对象如果判断失败应该如何提示用户。

# 感想
## `should.js`如何实现链式调用？
在`Assertion`类中，有一个`addChain`方法，该方法为某些属性定义了一些在getter函数中调用的操作方法，并且返回对象本身。通过这个方法，在`ext/chain.js`中，它为`should.js`中常见的语义词添加了属性，并通过返回对象本身来达到链式调用的`Assertion`对象传递。

	['an', 'of', 'a', 'and', 'be', 'has', 'have', 'with', 'is', 'which', 'the', 'it'].forEach(function(name) {
		Assertion.addChain(name);
	});

以下两段代码在结果上是一模一样的效果：		
	'1'.shoud.be.a.Number();
	'1'.should.be.be.be.be.a.a.a.a.Number();

## `should.js`的实现方式有哪些值得借鉴的地方？
1. `should.js`中，通过将一些语义词添加为属性值并返回`Assertion`对象本身，因此有效解决了链式调用的问题。
2. 通过`Asseriton`对象的属性来进行参数的传递，而不是通过函数参数，从而有效避免了函数调用时参数的传递问题以及多层调用时结构的复杂。
3. `should.js`通过扩展的方式来添加其判断的函数，保证了良好的扩展性，避免了代码耦合在一起，通过也为其他人编写更多的扩展代码提供了接口。
4. `should.js`通过extend方法，让`should(obj)`与`obj.should`两种方式达到了相同的效果。通过在`defineProperty`中定义should属性并且在回调函数中用`should(obj)`的方式来获取`obj`对象。
5. 通过抛出错误而不是返回布尔值的方式来通知用户，能够更加明显的通知用户，也方便向上抛出异常进行传递。

# 总结
总的来说，`should.js`是一个比较小而精的测试框架，他能够满足在开发过程中所需要的大部分测试场景，同时也支持自己编写扩展来强化它的功能。在设计上，这个框架使用了不少巧妙的方法，避免了一些复杂的链式调用与参数传递等问题，而且结构清晰，比较适合进行阅读与学习。


