# 背景

由于在项目中使用到了WebSocket的自定义二进制协议，需要将二进制转为后端服务中定义的Long型。而在JavaScript中的Number类型由于自身原因，并不能完全表示Long型的数字，因此需要我们通过其他的方式来对Long型值进行存储。

# 目标
在GitHub中，有一个实现了在JavaScript中存储Long型的对象，具体代码可以[戳此](https://github.com/dcodeIO/long.js)。下面，我们通过简单讲解一下这个库的具体实现来看看如何在JavaScript中实现一个Long型。如果你了解了这个实现原理，那么与之类似的，在JavaScript中实现一个Long Long型或者其他类型的方法也是类似的。

# 具体实现
其实，Long的实现很简单，我们现在只要回归到计算机的本质即可。在计算机中，其实存储的都是01字符串。例如，Int占4个字节(我们以32位操作系统为例)，而Long则占8个字节。

我们在存储中只需要将数据通过二进制进行存储，然后在操作中对二进制进行操作即可。

下面我们简单的来介绍一下Long的各个代表操作和思想。
## 大致步骤
### 数据存储
在Long型对象中，我们采用了高32位和低32位，以及加上一个符号位判断的值，用来进行数据的存储，具体格式如下：

```JavaScript
function Long(low, high, unsigned) {
    this.low = low | 0;
    this.high = high | 0;
    this.unsigned = !!unsigned;
}
```

通过对高低位的存储，从而让两个Number来同时表示一个Long型的高位和低位，从而满足了数值的长度要求。

### 转换为Long型
我们目前只介绍一个通过字符串来讲数据从String型转换为Long型，其他的转换例如从Number转换为Long型是类似的，我们就不过多赘述了。

先看实现函数：

```javascript
function fromString(str, unsigned, radix) {
	// 处理异常情况
    if (str.length === 0)
        throw Error('empty string');
        
    //处理为0的情况
    if (str === "NaN" || str === "Infinity" || str === "+Infinity" || str === "-Infinity")
        return ZERO;
        
    //处理只有两个参数的情况
    if (typeof unsigned === 'number') { 
        // For goog.math.long compatibility
        radix = unsigned,
        unsigned = false;
    } else {
        unsigned = !! unsigned;
    }
    radix = radix || 10;
    if (radix < 2 || 36 < radix)
        throw RangeError('radix');

    var p;
    if ((p = str.indexOf('-')) > 0)
        throw Error('interior hyphen');
    else if (p === 0) {
        // 转为正值处理
        return fromString(str.substring(1), unsigned, radix).neg();
    }
    
    // 从最高位分8位处理一次，如果长度超过8位，则先处理高位，然后将高位直接乘以进制的8次方，再处理低后8位，循环到最后8位为止
    var result = ZERO;
    for (var i = 0; i < str.length; i += 8) {
        var size = Math.min(8, str.length - i),
            value = parseInt(str.substring(i, i + size), radix);
        if (size < 8) {
            var power = fromNumber(pow_dbl(radix, size));
            result = result.mul(power).add(fromNumber(value));
        } else {
            result = result.mul(radixToPower);
            result = result.add(fromNumber(value));
        }
    }
    result.unsigned = unsigned;
    return result;
}
```
下面我们简单的说下这个函数的实现：

1. 对数据进行异常处理，排除一些边界条件。
2. 如果字符串为一个带"-"号的值，则转换为正值进行处理。
3. 如果字符串为一个常规的Long型值，则先从最前面的8位开始处理，将其通过指定的进制转换为Long型的值。
4. 处理接下来的8位，并且将之前的结果乘以进制数的8次方，即数字高地位的合并。例如：18 = 1 * 10^1 + 8。
5. 循环上面的操作，直到剩余的字符串长度小于8为止，即可结束，得到转换之后的Long型。

### 转换为字符串
Long型转换为字符串的方式，与字符串转换为Long型的步骤差不多，差不多是一个相反的过程。

```javascript
LongPrototype.toString = function toString(radix) {
    radix = radix || 10;
    if (radix < 2 || 36 < radix)
        throw RangeError('radix');
    if (this.isZero())
        return '0';
    //如果是负值,Unsigned型的Long值永远不会为负值
    if (this.isNegative()) {
        if (this.eq(MIN_VALUE)) {
            // We need to change the Long value before it can be negated, so we remove
            // the bottom-most digit in this base and then recurse to do the rest.
            var radixLong = fromNumber(radix),
                div = this.div(radixLong),
                rem1 = div.mul(radixLong).sub(this);
            return div.toString(radix) + rem1.toInt().toString(radix);
        } else
            return '-' + this.neg().toString(radix);
    }

    //每次处理6位，处理方式与字符串转换过来是类似的，和数学中十进制转换为N进制方法相同——相除法
    // Do several (6) digits each time through the loop, so as to
    // minimize the calls to the very expensive emulated div.
    var radixToPower = fromNumber(pow_dbl(radix, 6), this.unsigned),
        rem = this;
    var result = '';
    while (true) {
        var remDiv = rem.div(radixToPower),
            intval = rem.sub(remDiv.mul(radixToPower)).toInt() >>> 0,
            digits = intval.toString(radix);
        rem = remDiv;
        if (rem.isZero())
            return digits + result;
        else {
            while (digits.length < 6)
                digits = '0' + digits;
            result = '' + digits + result;
        }
    }
};
```

上面这个函数的实现步骤正好相反：

1. 处理各种边界条件
2. 如果Long型为一个负值，则转换为正值进行处理，如果Long型为`0x80000000`时，则对它进行了单独处理。
3. 在处理正值Long型为字符串时，操作方法与我们数学中教的转换进制的相除法类似，具体操作为：先除以需要转换的进制数，得到结果和余数，将结果重新作为被除数相除直到被除数为0，再将余数拼接起来即可。例如：18(10进制)转换为8进制时，操作是：`18 = 2 * 8 + 2; 2 = 0 * 8 + 2;`,因此结果为0x22。只是，在此函数中，一次相除的是进制数的6次方，其余步骤是类似的。
4. 通过上面的操作得到字符串后返回即可。

### Long型相加
在知道了Long型的存储本质是使用高低各32位以后，Long型的运算其实就已经了解了。我们只需要针对特定的操作进行相对应的二进制操作，那么我们就能够得到相对应的结果，下面的实例是Long型相加的操作，我们简单了解下：

```javascript
LongPrototype.add = function add(addend) {
    if (!isLong(addend))
        addend = fromValue(addend);
    // 将每个数字分成4个16比特的块，然后将这些块加起来

    var a48 = this.high >>> 16;
    var a32 = this.high & 0xFFFF;
    var a16 = this.low >>> 16;
    var a00 = this.low & 0xFFFF;

    var b48 = addend.high >>> 16;
    var b32 = addend.high & 0xFFFF;
    var b16 = addend.low >>> 16;
    var b00 = addend.low & 0xFFFF;

    var c48 = 0, c32 = 0, c16 = 0, c00 = 0;
    c00 += a00 + b00;
    c16 += c00 >>> 16;
    c00 &= 0xFFFF;
    c16 += a16 + b16;
    c32 += c16 >>> 16;
    c16 &= 0xFFFF;
    c32 += a32 + b32;
    c48 += c32 >>> 16;
    c32 &= 0xFFFF;
    c48 += a48 + b48;
    c48 &= 0xFFFF;
    return fromBits((c16 << 16) | c00, (c48 << 16) | c32, this.unsigned);
};
```

通过上面的操作我们就可以知道，Long型的四则运算等操作其实都是通过二进制和位运算来实现的。并没有我们想象中的那么神秘。

# 总结
其实，通过阅读`Long.js`库的源码你就会发现，在JavaScript中实现一个Long型并不难，也许还是一个听简单的事情，不过重要的是我们可能想象不到这种的实现方式。因此，这个也证明了我们在思考一个问题问题的同时，我们也应该多从事情的本质来考虑，这样就有可能得到解决方案。

# 附录
- 我在`Long.js`的代码中添加了一些中文的注释，如果有需要可以到我folk的[仓库](https://github.com/HJava/long.js)进行阅读学习。