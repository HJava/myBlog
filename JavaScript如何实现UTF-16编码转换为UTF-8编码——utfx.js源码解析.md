# 概述

当你在前端需要通过二进制数据与服务端进行通信时，你可能会遇到二进制数据的编码问题。大部分服务端的字符串编码类型都为UTF-8，而JavaScript中字符串编码类型是UTF-16，因此，你需要一个能够将字符串在两种编码方式间进行转换的方法。

本文通过对[utfx.js](https://github.com/dcodeIO/utfx)这个库的代码进行分析，带大家深入了解UTF8和UTF16这两种编码方式在JavaScript中的转换方法，同时加深对Unicode中UTF-8和UTF-16两种编码方式的具体原理的理解。

本文的主要内容为：

- utfx.js API简单介绍
- UTF-16编码转换为UTF-8编码
- UTF-8编码字符串长度计算
- 实验性功能：window.TextEncoder

如果有读者不了解Unicode中UTF-8和UTF-16两种编码方式的具体原理，可以阅读我的前一篇博客——[Unicode中UTF-8与UTF-16编码详解](https://juejin.im/post/5ace27c96fb9a028dc416195)。

如果有读者想要了解该库相关的转换使用场景，可以阅读我之前的博客[WebSocket系列之JavaScript字符串如何与二进制数据间进行互相转换](https://juejin.im/post/5abdc38ef265da2375070008)。

# utfx.js API简介

在进行具体的代码详解之前，我们先来了解下我们需要介绍的库——utfx.js。我们只有了解了这个库的使用方法，我们才能够更好的理解源码。

utfx.js代码不多，一共只有八个API接口，分别为：

- encodeUTF8：将UTF-8编码的字符串code码转换为二进制bytes。
- decodeUTF8：将UTF-8编码的二进制bytes解码城字符串code码。
- UTF16toUTF8：将UTF-16的字符转换为UTF-8的code码。
- UTF8toUTF16：将UTF-8的code码转换为UTF-16的字符。
- encodeUTF16toUTF8：将UTF-16编码的字符转换为UTF-8编码的bytes。
- decodeUTF8toUTF16：将UTF-8编码的bytes转换为UTF-16编码的字符。
- calculateCodePoint：计算UTF-8编码下的字符长度。
- calculateUTF8：计算需要用来存储UTF-8编码code码的bytes的长度。
- calculateUTF16asUTF8：计算UTF-16编码的字符在转换成UTF-8后需要的存储长度。

下面，我们将挑选几个具有代表性的API，针对其实现的具体代码来进行分析，帮助大家快速理解这两种编码方式。

# UTF-16编码转换为UTF-8编码

下面让我们来看下如何将UTF-16编码的数据转换为UTF-8编码的数据。

当我们需要把UTF-16的数据转换为UTF-8编码的数据时，最好的方法肯定是将UTF-16编码的数据转换为通用的Unicode码，在进行UTF-8编码。我们通过UTF16toUTF8和encodeUTF8方法的代码来进行具体解析。

## UTF16toUTF8

这个函数名看上去是直接将UTF-16编码的bytes数据转换为UTF-8编码的的Bytes数据。其实是，**将UTF-16编码的bytes数据转换为Unicode对应的二进制数据**。

```javascript
/**
 * UTF16数据转换到Unicode数据
 * @param src 数据源，类型为Function，调用一次返回1 Byte数据，如果到达字符串末尾则返回null
 * @param dst 处理函数，类型为Function，得到的Bytes作为参数传递给dst函数
 */
utfx.UTF16toUTF8 = function (src, dst) {
    var c1, c2 = null;
    while (true) {
        // 到达结尾调用src函数得到null后会进入此分支逻辑
        if ((c1 = c2 !== null ? c2 : src()) === null)
            break;
        
        //Unicode标准规定，U+D800~U+DFFF的值不对应任何字符，即专门用来判断是否为高位代理
        if (c1 >= 0xD800 && c1 <= 0xDFFF) {
            if ((c2 = src()) !== null) {
                // 如果Unicode码范围超过U+FFFF则会进入此分支逻辑（两段：第一段大于U+D800，第二段大于U+DC00）
                if (c2 >= 0xDC00 && c2 <= 0xDFFF) {
                    // 第一步：用c1还原高10位；第二步：用c2还原低十位；第三步：加上减去的0x10000
                    dst((c1 - 0xD800) * 0x400 + c2 - 0xDC00 + 0x10000);
                    c2 = null; continue;
                }
            }
        }
        dst(c1);
    }
    if (c2 !== null) dst(c2);
};
```

根据代码和上面的注释，大家应该就能看懂对应代码，因此在此不做过多赘述。我们接着看将Unicode码转换为UTF-8编码的方法。

## encodeUTF8

该方法是将Unicode码进行UTF-8编码转换，从而得到UTF-8编码的Bytes数据。

```javascript
/**
 * Unicode数据转换为UTF-8数据
 * @param src 数据源，类型为Function，调用一次返回1 Byte数据，如果到达字符串末尾则返回null
 * @param dst 处理函数，类型为Function，得到的Bytes作为参数传递给dst函数
 */
utfx.encodeUTF8 = function (src, dst) {
    var cp = null;
    if (typeof src === 'number')
        cp = src,
            src = function () {return null;};
    while (cp !== null || (cp = src()) !== null) {
        if (cp < 0x80)
        // 1 byte存储情况
            dst(cp & 0x7F);
        else if (cp < 0x800)
        // 2 byte存储情况
            dst(((cp >> 6) & 0x1F) | 0xC0),
            dst((cp & 0x3F) | 0x80);
        else if (cp < 0x10000)
        // 3 byte存储情况
            dst(((cp >> 12) & 0x0F) | 0xE0),
            dst(((cp >> 6) & 0x3F) | 0x80),
            dst((cp & 0x3F) | 0x80);
        else
        // 4 byte存储情况
            dst(((cp >> 18) & 0x07) | 0xF0),
            dst(((cp >> 12) & 0x3F) | 0x80),
            dst(((cp >> 6) & 0x3F) | 0x80),
            dst((cp & 0x3F) | 0x80);
        cp = null;
    }
};
```

上面的代码与UTF-8编码规范中的方式基本一致，如果没有理解相关规范，可以先阅读本文概述中提到的前一篇博客。

# 编码字符串长度计算

当我们给出一串Unicode码时，我们需要知道申请多大的ArrayBuffer来进行转换后的数据存储。正好，这个库还提供了根据Unicode码的长度或者UTF-16编码格式的数据来计算UTF-8数据的存储长度。

下面我们来介绍`calculateUTF8`和`calculateUTF16asUTF8`这两个方法。

## calculateUTF8

该方法是通过Unicode码来计算转换为UTF-8编码后所占存储长度。

```javascript
/**
 * 根据Unicode编码来计算转换成UTF-8编码后需要的存储长度
 * @param src 数据源，类型为Function，调用一次返回1 Byte数据，如果到达字符串末尾则返回null
 */
utfx.calculateUTF8 = function (src) {
    var cp, l = 0;
    while ((cp = src()) !== null)
        // 占1 Byte的范围是0~0x7F；占2 Byte的范围是0x80~0x7FF；占三个字节的范围是0x800~0xFFFF；占4个字节的范围为：0x10000~0x10FFFF
        l += (cp < 0x80) ? 1 : (cp < 0x800) ? 2 : (cp < 0x10000) ? 3 : 4;
    return l;
};
```

根据上面的的代码和UTF-8的编码规范，我们就能够很容易理解这种宽度计算的方法。

## calculateUTF16asUTF8

该方法是通过UTF16的数据来计算转换为Unicode码和转换为UTF-8编码后所占存储长度。

```javacript
/**
 * 根据UTF-16编码的Bytes来计算转换为Unicode的长度和转换成UTF-8编码后需要的存储长度
 * @param src 数据源，类型为Function，调用一次返回1 Byte数据，如果到达字符串末尾则返回null
 */
utfx.calculateUTF16asUTF8 = function (src) {
    var n = 0, l = 0;
    utfx.UTF16toUTF8(src, function (cp) {
        ++n; l += (cp < 0x80) ? 1 : (cp < 0x800) ? 2 : (cp < 0x10000) ? 3 : 4;
    });
    return [n, l];
};
```

该方法通过之前介绍的将UTF-16编码转换为Unicode码的方法获取到Unicode数据，再进行计算，返回了Unicode码的长度和UTF-8编码后长度。

# window.TextEncoder与Window.TextDecoder

这是两个处在实验性的新构造函数，通过创建编码器（`TextEncode`对象）和解码器（`TextDecode`对象）来实现JavaScript中string类型与UTF-8编码数据中的互相转换。

构造方法将会返回一个UTF-8编码的，使用方法如下：

```javascript
let encoder = new TextEncoder();
let decoder = new TextDecoder();

let unit8Array = encoder.encode('a'); // 返回一个Unit8Array类型——[97]
let str = decoder.decode(arr); // 返回一个值为'a'的字符串
```

目前，这项新技术的的兼容性仍然存在很大问题，只有Chrome 38、Firefox 19以及Opera 25以上才支持，其他主流的浏览器如IE和Safari都还没有任何支持，因此在生产环节中需要慎重使用。

# 总结

本文对实现了Unicode中UTF-8和UTF-16这两种编码方式的库——[utfx.js](https://github.com/dcodeIO/utfx)进行了部分代码分析。通过看到具体的代码实现，相信大家应该能够更加好的理解这两种编码方式的具体规范，以及对应的使用方式和场景。



