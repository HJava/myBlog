# 概述
上一篇博客我们说到了如何进行数字类型（如Short、Int、Long类型）如何在JavaScript中进行二进制转换，如果感兴趣的可以可以阅读本系列第二篇博客——[WebSocket系列之JavaScript中数字数据如何转换为二进制数据](https://juejin.im/post/5abb560a6fb9a028d141262b)。这次，我们来说下string类型的数据如何进行处理。
本文是WebSocket系列的第三篇，主要介绍string数据与二进制数据之间的转换方法，具体的内容如下：
- JavaScript中string类型基础知识
- JavaScript如何将string类型转换为二进制数据
- JavaScript如何将二进制数据转换为string类型
  本文与WebSocket并无太强关联，不过作为在WebSocket中传递二进制数据的基础知识储备，因此放入了此系列当中。
  如果读者对WebSocket并不了解，或者说不明白它的使用场景和细节，可以阅读我的本系列的第一篇博客——[WebSocket系列之基础知识入门篇](https://juejin.im/post/5ab91ac96fb9a028db58b1d5)。
# string类型基础知识
string这个类型，对于熟悉JavaScript的同学应该都不陌生，它是属于JavaScript中基础数据类型的一种。不过，我们今天要先介绍下`DOMString`。
> DOMString 是一个UTF-16字符串。由于JavaScript已经使用了这样的字符串，所以DOMString 直接映射到 一个String。将null传递给接受DOMString的方法或参数时通常会把其转换成为“null”。

在WebSocket中进行string类型数据传输时，使用的其实也是`DOMString`。不过，根据[MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/DOMString)中对`DOMString`的介绍我们可以了解到，大部分日常使用场景中，我们可以认为`DOMString`就是一个string类型。所以，我们只需要了解此类型，使用时仍然当成string类型处理即可。

## 编码
既然上面提到了UTF-16，那么我们就来简单介绍下UTF-16，以及后端常用的UTF-8这两种编码方式。
为什么需要介绍编码类型呢？因为我们在与后端进行字符串数据传递时，可能使用的编码方式不同，这样就会导致双方得到不同的数据。因此，我们在进行string的二进制数据通信时，不仅仅需要将字符串转换成二进制，还需要协商一致的string编码。
### UTF-16
> UTF-16 (16-bit Unicode Transformation Format) 是Unicode字符编码五层次模型的第三层：字符编码表（Character Encoding Form，也称为"storage format"）的一种实现方式。即把Unicode字符集的抽象码位映射为16位长的整数（即码元）的序列，用于数据存储或传递。
Unicode字符的码位，需要1个或者2个16位长的码元来表示，因此这是一个变长表示。

UTF-16是JavaScript中的string编码类型。

### UTF-8
> UTF-8（8-bit Unicode Transformation Format）是一种针对Unicode的可变长度字符编码，也是一种前缀码。它可以用来表示Unicode标准中的任何字元，且其编码中的第一个字节仍与ASCII兼容，这使得原来处理ASCII字元的软件无须或只须做少部分修改，即可继续使用。UTF-8使用一至四个字节为每个字符编码（2003年11月重新规范）。

UTF-8是很多语言使用的通用编码类型，在后端应用中非常常见。

### JavaScript中UTF16和UTF-8如何进行编码转换
在Github上有转换库[GitHub - dcodeIO/utfx: A compact library to encode, decode and convert UTF8 / UTF16 in JavaScript.](https://github.com/dcodeIO/utfx)，通过这个库，可以将字符串在UTF-8编码和UTF-16编码中进行转换。该库的具体原理和内容以及两种编码方式的详细内容说明将会在之后的博客中进行讲解。我们下面简单的介绍下它是使用方式：
```javascript
import utfx from './util/utfx';

let str = 'abcdefg';
let result = [];

function stringSource(s) {
    let i = 0;
    return function () {
        return i < s.length ? s.charCodeAt(i++) : null;
    };
}

utfx.encodeUTF16toUTF8(stringSource(str), function (b) {
    result.push(b);
}.bind(this));
```
同理，该类库提供了其他的方法：
- `decodeUTF8toUTF16`，将UTF-8的string数据转换为UTF-16的string数据。
- `calculateUTF16asUTF8`，计算UTF-16编码的string类型类型转换为UTF-8后所占Byte长度。
  这两个方法我们在之后的章节中也会用到。
# JavaScript如何将string类型转换为二进制数据
了解了JavaScript中string类型的编码和在UTF-8和UTF-16之间转换编码的方式，下面我们来看下如何将string类型转换为二进制数据。
首先，我们假定与后端交互时使用的编码方式为UTF-8，这样能够满足更多的使用场景。如果仍然使用UTF-16的话，则直接忽略转换编码的逻辑即可。
简单介绍下实现思路：我们得到一个需要转换的字符串后，先知道其长度后，初始化ArrayBuffer中相关参数，将数据放入ArrayBuffer中即可。我们将上面的示例稍作改动：
```javascript
import utfx from './util/utfx';

function stringSource(s) {
    var i = 0;
    return function () {
        return i < s.length ? s.charCodeAt(i++) : null;
    };
}

let str = 'abcdefg';

let strCodes = stringSource(str);
let length = utfx.calculateUTF16asUTF8(strCodes)[1];
let buffer = new ArrayBuffer(length + 4); // 初始化长度为UTF8编码后字符串长度+4个Byte的二进制缓冲区
let view = new DataView(buffer);
let offset = 4;

view.setUint32(0, length); // 将长度放置在字符串的头部

utfx.encodeUTF16toUTF8(stringSource(str), function (b) {
    view.setUint8(offset++, b);
}.bind(this));
```
通过上面的示例，我们就已经将一个二进制数据根据UTF-8编码后放入了ArrayBuffer中，同时，将其长度作为一个Unsigned Int类型存储在了二进制头部4个Byte的位置。
# JavaScript如何将二进制数据转换为string类型
知道了如何将string类型转换为二进制数据，下面我们看下如何将整个数据从二进制中读取，转换回string类型。
根据上面转换为二进制的过程，我们不难想到相关的二进制转string类型方法。具体示例如下：
```javascript
import utfx from './util/utfx';
let str = 'abcdefg';

function stringSource(s) {
    var i = 0;
    return function () {
        return i < s.length ? s.charCodeAt(i++) : null;
    };
}

let strCodes = stringSource(str);
let length = utfx.calculateUTF16asUTF8(strCodes)[1];
let buffer = new ArrayBuffer(length + 4); // 初始化长度为UTF8编码后字符串长度+4个Byte的二进制缓冲区
let view = new DataView(buffer);
let offset = 4;

// 字符串转换二进制过程
view.setUint32(0, length); // 将长度放置在字符串的

utfx.encodeUTF16toUTF8(stringSource(str), function (b) {
    view.setUint8(offset++, b);
}.bind(this));

// 二进制转换字符串过程
let Strlength = view.getUint32(0);
offset = 4;
let result = []; // Unicode编码字符
let end = offset + Strlength;


utfx.decodeUTF8toUTF16(function () {
    return offset < end ? view.getUint8(offset++) : null; // 返回null时会退出此转换函数
}.bind(this), (char) => {
    console.log(char)
    result.push(char);
});

let strResult = result.reduce((prev, next)=>{
    return prev + String.fromCharCode(next);
}, '');
```
通过上面的示例我们可以知道，我们只需要在前面4个Byte中将字符串长度读取出来，然后再从第4个Byte（从0开始算）的位置开始读取指定长度的字符串字符编码即可。最后，我们得到了一个Unicode码数组，只需要`fromCharCode`方法即可将其转换为字符串。
# 总结
通过使用ArrayBuffer和DataView，我们能够在string数据和二进制数据中进行互相转换。
有了string类型转换的相关基础，读者就能够在之后的WebSocket进行二进制数据传递时理解相关的内容和处理逻辑。
下一篇WebSocket系列相关的博客，将会介绍如何通过WebSocket来向后端传递二进制数据，以及如何处理通过WebSocket收到的二进制数据。有兴趣的同学可以继续关注。

