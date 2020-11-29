## 概述

在我们常见的JavaScript数字运算中，小数和大数都是会让我们比较头疼的两个数据类型。

- 在大数运算中，由于number类型的数字长度限制，我们经常会遇到超出范围的情况。比如在我们传递Long型数据的情况下，我们就只能把它转换到字符串进行传递和处理。
- 而在小数点数字进行运算的过程中，JavaScript又由于它的数据表示方式，从而导致了小数运算会有不准确的情况。最经典的一个例子就是0.3-0.2，并不等于0.1，而是等于0.09999999999999998。

在之前的博客中我介绍了一个Long类型数据处理的库，叫做[long.js](https://github.com/dcodeIO/Long.js/)，它能够比较有效地处理弄型数据。从而扩展JavaScript在数据类型中的一个处理能力，大家如果感兴趣的话可以去看一下这篇文章：[Long.js源码分析与学习](https://juejin.cn/post/6844903565291421703)。

今天我们需要介绍的这个库，它不仅仅能够支持处理Long类型的数据，也能够准确的处理小数的运算，他就是[big.js](https://github.com/MikeMcl/big.js/)。

这个库历史也很悠久了，从commit记录来看，第一次提交还是在2012年。现在，它已经拥有了3.2K个star，这足以表明这个库受欢迎的程度。同时，这个库具有完备的单元测试，目前所有的issue都已经解决，所以大家也不用担心这个库的质量问题。

## 代码示例

首先，让我们来看下，big.js这个库到底是如何使用的，具体有哪些应用的场景和功能。

```typescript
x = new Big(123.4567)
y = Big('123456.7e-3')                 // 'new' is optional
z = new Big(x)
x.eq(y) && x.eq(z) && y.eq(z)


0.3 - 0.1                              // 0.19999999999999998
x = new Big(0.3)
x.minus(0.1)                           // "0.2"
x                                      // "0.3"
```

通过代码，我们可以看到，big.js所有的操作都是基于Big类的。Big类实现了我们在数字运算中的一些常见的操作，例如加减乘除、比较等。基本上你用到的操作，应该都是支持了。

如果想要了解big.js具体支持哪些方法，可以阅读[big.js API文档](http://mikemcl.github.io/big.js/)。

## API简介

big.js的API主要分为以下两个部分：

- 常量定义
- 运算操作函数

接下来，我们一个一个部分来看。

#### 常量定义

big.js的常量定义一共有5个，分别的含义是：

- DP，小数点后位数，默认值是20
- RM，四舍五入方式，默认为1，代表向最近的整数取整。如果是0.5，那么向下取整。
- NE：在转换为字符串时展示为科学计数法的最小小数位数。默认值是-7，即小数点后第7为才开始不是0。
- PE：在转换为字符串时展示位科学计数法的最小整数位数。默认值是21，即数字长度超过21位。
- strict：默认值为false。设置为true时，构造函数只接受字符串和大数。

#### 运算符操作函数

- abs，取绝对值。
- cmp，compare的缩写，即比较函数。
- div，除法。
- eq，equal的缩写，即相等比较。
- gt，大于。
- gte，小于等于，e表示equal。
- lt，小于。
- lte，小于等于，e表示equal。
- minus，减法。
- mod，取余。
- plus，加法。
- pow，次方。
- prec，按精度舍入，参数表示整体位数。
- round，按精度舍入，参数表示小数点后位数。
- sqrt，开方。
- times，乘法。
- toExponential，转化为科学计数法，参数代表精度位数。
- toFied，补全位数，参数代表小数点后位数。
- toJSON和toString，转化为字符串。
- toPrecision，按指定有效位数展示，参数为有效位数。
- toNumber，转化为JavaScript中number类型。
- valueOf，包含负号（如果为负数或者-0）的字符串。

## 源码解析

big.js的源码内容比较少，都在big.js一个文件中，大家如果想要阅读，直接看这个文件就行。接下来让我们来看一下big.js的代码结构。

常量定义我们就不过多介绍了，这个代码没有什么复杂的。我们主要来看下内部的数据在初始化后如何进行存储的，以及我们选择几个特定的API，看下这些API是如何实现的。

#### 变量存储

我们首先来看下构造函数。

```typescript
function Big(n) {
    var x = this;

    // 支持函数调用方式进行初始化,可以不使用new操作符
    if (!(x instanceof Big)) return n === UNDEFINED ? _Big_() : new Big(n);

    // 原型链判断,确认传入值是否已经为Big类的实例
    if (n instanceof Big) {
        x.s = n.s;
        x.e = n.e;
        x.c = n.c.slice();
    } else {
        if (typeof n !== 'string') {
            if (Big.strict === true) {
                throw TypeError(INVALID + 'number');
            }

            // 确定是否为-0,如果不是,转化为字符串.
            n = n === 0 && 1 / n < 0 ? '-0' : String(n);
        }

        // parse函数只接受字符串参数
        parse(x, n);
    }

    x.constructor = Big;
}
```

在构造函数中，传入的变量n经过各种类型判断，最终变成了一个字符串，传给了`parse`函数。那么，我们接下来看看`parse`函数做了什么。

```typescript
function parse(x, n) {
    var e, i, nl;

    if (!NUMERIC.test(n)) {
        throw Error(INVALID + 'number');
    }

    // 判断符号,是正数还是负数
    x.s = n.charAt(0) == '-' ? (n = n.slice(1), -1) : 1;

    // 判断是否有小数点
    if ((e = n.indexOf('.')) > -1) n = n.replace('.', '');

    // 判断是否为科学计数法
    if ((i = n.search(/e/i)) > 0) {

        // 确定指数值
        if (e < 0) e = i;
        e += +n.slice(i + 1);
        n = n.substring(0, i);
    } else if (e < 0) {

        // 是一个正整数
        e = n.length;
    }

    nl = n.length;

    // 确定数字前面有没有0,例如0123这种0
    for (i = 0; i < nl && n.charAt(i) == '0';) ++i;

    if (i == nl) {

        // Zero.
        x.c = [x.e = 0];
    } else {

        // 确定数字后面的0,例如1.230这种0
        for (; nl > 0 && n.charAt(--nl) == '0';);
        x.e = e - i - 1;
        x.c = [];

        // 把字符串转换成数组进行存储,这个时候已经去掉了前面的0和后面的0
        for (e = 0; i <= nl;) x.c[e++] = +n.charAt(i++);
    }

    return x;
}
```

在`parse`函数中，进行了数据的解析处理。首先，我们判断了是否符合数字的标准，如果符合的话，我们对传入的数据表示的数字方法进行了判断，是不是负数、是不是小数、有没有适用科学计数法，同时对一些无意义的0进行了处理。

通过`parse`函数，我们就已经把构造函数传入的数据转化成了Big类的实例中的属性了。其中，Big类实例中的变量，存储的值对应的含义如下：

- s，表示符号，`-1`表示负数，`1`表示正数。
- c，是一个数组，存储了当前数字的每一位的值。
- e，表示小数的开始位数，即在数组中的第几个元素是小数的开始。比如[1,2,3,4]中，如果`e`是2，那么就代表着12.34。

通过上述的存储结构描述，大家应该对big.js的数据存储方式有了一个清楚的了解。大家应该也能够大概猜到，接下来的一些运算函数，应该如何对这些数据进行操作了。接下来，我们挑选几个最常见的函数，验证下我们的想法是不是准确的。

#### API源码解析

因为big.js支持的运算比较多，因此我们就选几个比较有代表性的，其他的大家感兴趣，可以自己顺着源码看下，整体上还是很好理解的。

**加法**

首先我们来看下四则运算中最简单的加法，这个我们简单思考下就很简单了，只需要判断下符号，对应上位数进行加减运算即可。我们看下具体的实现代码。

```typescript
P.plus = P.add = function (y) {
    var t,
        x = this,
        Big = x.constructor,
        a = x.s,
        // 所有操作均转化为两个Big类的实例进行运算,方便处理
        b = (y = new Big(y)).s;

    // 判断符号是不是不相等,即一个为正,一个为负
    if (a != b) {
        y.s = -b;
        return x.minus(y);
    }

    var xe = x.e,
        xc = x.c,
        ye = y.e,
        yc = y.c;

    // 判断是否某个值是0
    if (!xc[0] || !yc[0]) return yc[0] ? y : new Big(xc[0] ? x : a * 0);

    // 拷贝一份数组,避免影响原实例
    xc = xc.slice();

    // 填0来保证运算时的位数相等
    // 注意,reverse函数比unshift函数快
    if (a = xe - ye) {
        if (a > 0) {
            ye = xe;
            t = yc;
        } else {
            a = -a;
            t = xc;
        }

        t.reverse();
        for (; a--;) t.push(0);
        t.reverse();
    }

    // 把xc放到一个更长的数组中,方便后续循环加法操作
    if (xc.length - yc.length < 0) {
        t = yc;
        yc = xc;
        xc = t;
    }

    a = yc.length;

    // 执行加法操作,将数值保存到xc中
    for (b = 0; a; xc[a] %= 10) b = (xc[--a] = xc[a] + yc[a] + b) / 10 | 0;

    // 不需要检查0,因为 +x + +y != 0 ,同时 -x + -y != 0

    if (b) {
        xc.unshift(b);
        ++ye;
    }

    // 删除结尾的0
    for (a = xc.length; xc[--a] === 0;) xc.pop();

    y.c = xc;
    y.e = ye;

    return y;
};
```

通过上述的代码中，我们可以看到，加法操作其实就是在符号相同的情况下，对齐两个数字的小数点，然后对数组中的每一对数据进行加法操作，得到结果后再保存下来。这个算是一个比较简单的操作。

在代码中，big.js通过一些边界条件判断和交换的方法，保证了运算流程的简单。

**乘法**

看完了简单的加法，我们来看下稍微复杂一些的乘法。其实乘法的本质和加法也是类似的，每一位数字进行运算后再保存回原数组即可。想想我们小学学过的乘法计算方式，那么就不难理解这个代码。

```typescript
P.times = P.mul = function (y) {
    var c,
        x = this,
        Big = x.constructor,
        xc = x.c,
        yc = (y = new Big(y)).c,
        a = xc.length,
        b = yc.length,
        i = x.e,
        j = y.e;

    // 符号比较确定最终的符号是为正还是为负
    y.s = x.s == y.s ? 1 : -1;

    // 如果有一个值是0,那么返回0即可
    if (!xc[0] || !yc[0]) return new Big(y.s * 0);

    // 小数点初始化为x.e+y.e,这是我们在两个小数相乘的时候,小数点的计算规则
    y.e = i + j;

    // 这一步也是保证xc的长度永远不小于yc的长度,因为要遍历xc来进行运算
    if (a < b) {
        c = xc;
        xc = yc;
        yc = c;
        j = a;
        a = b;
        b = j;
    }

    // 用0来初始化结果数组
    for (c = new Array(j = a + b); j--;) c[j] = 0;

    // i初始化为xc的长度
    for (i = b; i--;) {
        b = 0;

        // a是yc的长度
        for (j = a + i; j > i;) {

            // xc的一位乘以yc的一位,得到最终的结果值,保存下来
            b = c[j] + yc[i] * xc[j - i - 1] + b;
            c[j--] = b % 10;

            b = b / 10 | 0;
        }

        c[j] = b;
    }

    // 如果有进位,那么就调整小数点的位数(增加y.e),否则就删除最前面的0
    if (b) ++y.e;
    else c.shift();

    // 删除后面的0
    for (i = c.length; !c[--i];) c.pop();
    y.c = c;

    return y;
};
```

乘法的整体思路也是类似，先确定符号，然后调整小数点位数，最后进行乘法运算得到最终的结果。整体的代码看上去还是很清晰。

**取整**

看完了四则运算中有代表的加法和乘法，我们来看下取整这个运算。

在big.js中，所有的取整运算都调用了内部的一个`round`函数。那么，接下来，我们就以API中的`round`方法为例。这个方法有两个参数，第一个值`dp`代表着小数后有效值的位数，第二个`rm`代表了取整的方式。

```typescript
P.round = function (dp, rm) {
    if (dp === UNDEFINED) dp = 0;
    else if (dp !== ~~dp || dp < -MAX_DP || dp > MAX_DP) {
        throw Error(INVALID_DP);
    }
    return round(new this.constructor(this), dp + this.e + 1, rm);
};

function round(x, sd, rm, more) {
    var xc = x.c;

    if (rm === UNDEFINED) rm = Big.RM;
    if (rm !== 0 && rm !== 1 && rm !== 2 && rm !== 3) {
        throw Error(INVALID_RM);
    }

    if (sd < 1) {
        // 兜底情况,精度小于1,默认有效值为1
        more =
            rm === 3 && (more || !!xc[0]) || sd === 0 && (
                rm === 1 && xc[0] >= 5 ||
                rm === 2 && (xc[0] > 5 || xc[0] === 5 && (more || xc[1] !== UNDEFINED))
            );

        xc.length = 1;

        if (more) {

            // 1, 0.1, 0.01, 0.001, 0.0001 等等
            x.e = x.e - sd + 1;
            xc[0] = 1;
        } else {
            // 定义为0
            xc[0] = x.e = 0;
        }
    } else if (sd < xc.length) {

        // xc数组中,在精度之后的纸会被舍弃取整
        more =
            rm === 1 && xc[sd] >= 5 ||
            rm === 2 && (xc[sd] > 5 || xc[sd] === 5 &&
                (more || xc[sd + 1] !== UNDEFINED || xc[sd - 1] & 1)) ||
            rm === 3 && (more || !!xc[0]);

        // 删除所需精度后的数组值
        xc.length = sd--;

        // 取整方式判断
        if (more) {

            // 四舍五入可能意味着前一个数字必须四舍五入,所以这个时候需要填0
            for (; ++xc[sd] > 9;) {
                xc[sd] = 0;
                if (!sd--) {
                    ++x.e;
                    xc.unshift(1);
                }
            }
        }

        // 删除小数点后面的0
        for (sd = xc.length; !xc[--sd];) xc.pop();
    }

    return x;
}
```

通过内部的`round`函数的实现可以看到，在最开始我们进行了异常的兜底检测，排除了两种异常的情况。一种是参数错误，直接抛出异常；另一种是精度小于1的情况，在这个时候，定义了兜底的值1。

在正常的逻辑中，我们根据精度舍弃了精度后的值，统一填充0进行表示。

#### 源码解析小结

在big.js的源码中，我们看到了大数的处理方式——通过将大数拆解成每一位，然后进行每一位运算，得到结果。

还有一些我们没有提到过的运算操作，例如取绝对值、减法、除法、次方等，原理都很简单。他们都是通过操作我们存储在实例中的三个属性：符号、小数点位和数字的字符串，来获取最终的结果。这个由于篇幅原因，我们就不过多赘述了，大家有兴趣的可以去看看源码。

我们来看下在源码中，有哪些优秀的地方值得我们学习：

- 大数的处理方式。通过源码阅读能够使我们更加明确数字的运算方法。
- 处理顺序统一。在每一个运算函数中，我们都是先进行异常检测，然后对数据进行处理，最终，我们定义了统一的处理逻辑，对数据进行运算操作。如果遇到不符合这种处理逻辑的数值，我们都在处理前被转化成了符合要求的值。这样，我们的代码看起来条理清晰，思路明确，不需要通过不同的逻辑处理代码来处理不同类型的数据。

但是，在代码中，其实我们也发现了一些小的瑕疵，比如常量定义使用的数字，而不是更加便于理解的常量或者字符串，这个其实是可以再进行优化的。

## 总结

总体上来说，big.js是一个非常精简的库。它的源码还是比较便于理解的。这个方式比之前的long.js来说，操作更加的简单，看上去也更加的通俗易懂。

在最新的JavaScript中，也支持了大整型的数据类型——[BigInt](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/BigInt)。这个能够覆盖在整型数字超过Number类型时的一些运算和处理，有兴趣的同学也可以去看看。

总体上来说，我还是推荐大家使用像big.js这种库对大数进行处理，一个是能够保证各平台兼容性，不存在跨平台和容器高低版本问题，另一个是数字数据类型统一，方便后续统一处理（BigInt和Number类型不可一起运算，BigInt不支持Math对象中的方法）。

如果大家后续需要对大数进行操作，可以考虑使用这个精简又方便的库。

如果大家希望看有中文注释的代码，也可以去我的[GitHub中看我folk的代码](https://github.com/HJava/big.js)，里面有部分中文，后续也会补齐。