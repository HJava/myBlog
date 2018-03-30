# 概述

本文主要通过对JavaScript中数字数据与二进制数据之间的转换，让读者能够了解在JavaScript中如何对数字类型（包括但不限于Number类型）进行处理。

二进制数据在日常的JavaScript中很少遇到，但是当你使用WebSocket与后端进行数据交互时，就有可能会用到二进制的数据格式。因此，为了更好的理解本系列中之后发布的关于WebSocket传输二进制相关的内容，我们有必要了解二进制数据在JavaScript中是如何进行操作和存储的。

本文内容主要为：

- JavaScript中如何操作与存储二进制数据——ArrayBuffer存储结构相关基础知识以及对应的DataView相关数据类型基础知识和和API接口，同时对字节序问题进行介绍。
- 以Int和Short为例，说明JavaScript中的数字数据如何转换为二进制数据。
- 以Long类型为例，说明JavaScript中如何表示Long类型并且如何将其转换为二进制数据。
- 如何将二进制数据中转换为JavaScript中的数字数据。

本文与WebSocket并无太强关联，不过作为在WebSocket中传递二进制数据的基础知识储备，因此放入了此系列当中。

如果读者对WebSocket并不了解，或者说不明白它的使用场景和细节，可以阅读我的前一篇博客——[WebSocket系列之基础知识入门篇](https://juejin.im/post/5ab91ac96fb9a028db58b1d5)。

如果读者想了解String类型与二进制之间的处理和转换，可以于都WebSocket系列稍后发布的文章（文章发布后会替换此段）。

如果读者想了解在WebSocket中如何进行二进制的传递和解析，可以阅读WebSocket系列稍后发布的文章（文章发布后会替换此段）。

# JavaScript中如何存储和操作二进制数据

了解了为什么需要使用二进制数据，我们来看下，在JavaScript中如何存储和操作二进制数据。

## ArrayBuffer

首先，我们要介绍下在JavaScript中用来存储二进制数据的`ArrayBuffer`。

> **ArrayBuffer** 对象用来表示通用的、固定长度的原始二进制数据缓冲区。

在[MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer)的文档中，我们能够看到`ArrayBuffer`的介绍。它是在JavaScript中用来进行二进制数据存储的一种数据对象。

下面我们通过一个示例来简单介绍下ArrayBuffer相关操作。

```javascript
const buffer = new ArrayBuffer(8);

buffer.byteLength; // 结果为8
```

上面的示例通过创建一个长度为8Byte的二进制数据缓冲区。缓冲区只是一个数据存储的空间，如何对这个存储空间进行读取，完全取决于使用者。例如：8个字节可以当成是2个Int类型的数据，也可以是一个Long类型的数据，或者4个Short型的数据。

### DataView

看完了存储数据的`ArrayBuffer`，我们来看下数据读写的`DataView`。

> **DataView** 视图是一个可以从 [`ArrayBuffer`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer) 对象中读写多种数值类型的底层接口，在读写时不用考虑平台字节序问题。

这个是在[MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/DataView)中关于DataView的介绍。DataView提供了大量的API接口来进行数据的读和写操作，我们在后三章将会举例进行说明。但是，首先我们得先看下说明中提到的字节序问题。

### 字节序

在现有的计算机体系中，有两种字节序：

- 大端字节序：高位在前，低位在后。符合人类阅读习惯。
- 小端字节序：低位在前，高位在后。符合计算机读取习惯。

上面所说的顺序均是针对多字节对象而言，如Int类型，Long类型。以Int类型数据`0x1234`为例，如果是大端字节序，那么数据从人类对数值的通常写法上来看就是`0x1234`；如果是小端字节序，那么从人类对数值的通常写法上来看，应该写成`0x3412`。

对于单字节对象如Byte类型数据而言，没有字节序一说。

在不同的平台中，可能使用不同的字节序，这就是所谓的字节序问题。DataView所谓的在读写时不需要考虑平台字节序问题是指：同时使用DataView进行写入和读取的数据保持一致。

# JavaScript中的数字数据如何转换为二进制数据

对ArrayBuffer和DataView有了一个大概的了解，下面让我们来看下它是如何进行二进制数据操作的。

本章，我以Short类型和Int类型为例，介绍下相关操作步骤。

```javascript
let buffer = new ArrayBuffer(6); // 初始化3个Byte的二进制数据缓冲区
let dataView = new DataView(buffer);

dataView.setInt16(0, 3); // 从第0个Byte位置开始，放置一个数字为3的Short类型数据(占2 Byte)
dataView.setInt32(2, 15); // 从第2个Byte位置开始，放置一个数字为15的Short类型数据(占4 Byte)
```

通过上面的示例，我们一共初始化了6个Byte的存储空间，使用1个Short类型（占2 Byte）和一个Int类型（占4 Byte）的数据进行填充。

DataView还提供了许多的API接口来进行其他数据类型的处理，如无符号型，浮点数等。他们的使用方法和上面介绍的API相同，我们在这里就不一一进行介绍了，希望了解更多API接口的读者可以查看[MDN文档](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/DataView)。

# JavaScript中如何表示Long类型并且如何将其转换为二进制数据

通过DataView提供的API接口，我们知道了如何处理Short类型、Int类型、Float类型和Double类型。那么，如果是对于Long类型这种原生API中没有提供处理函数的数据类型，我们应该如何处理呢？

首先，我们需要理解Long数据类型的结构，它是由一个高位的4个Byte和低位的4个Byte组成的数据类型。因为Long类型表示的范围比Number类型大，所以我们在JavaScript中是使用了两个Number类型（即Int类型）的对象来表示Long类型数据，相关的具体细节可以见我之前的博客[Long.js源码分析与学习](https://juejin.im/post/5a88e148f265da4e761fd400)。

理解了JavaScript中如何存储Long类型，我们就知道如果对其进行存储。

```javascript
import Long from 'long';

let long = Long.fromString('123');
let buffer = new ArrayBuffer(8);
let dataView = new DataView(buffer);

dataView.setInt32(0, long.high); // 采用大端字节序放置
dataView.setInt32(4, long.low);
```

通过上面的示例，我们将一个Long类型的数据拆分成了两个Int类型的数据，按照大端字节序放入到了ArrayBuffer中。同理，如果是想按照小端字节序放置，只需要将数据进行部分处理后再放入即可，在此我就不过多介绍了。

# 如何将二进制数据中转换为JavaScript中的数据类型

当你知道了如何将数据转换为ArrayBuffer中存储的二进制数据后，就能够简单推测出如何进行反向操作——将数据从ArrayBuffer中读取出来，再转换成JavaScript中常用数据类型。

```javascript
import Long from 'long';

let buffer = new ArrayBuffer(14); // 初始化3个Byte的二进制数据缓冲区
let dataView = new DataView(buffer);
let long = Long.fromString('123');


// 数据写入过程

dataView.setInt16(0, 3); // 从第0个Byte位置开始，放置一个数字为3的Short类型数据(占2 Byte)
dataView.setInt32(2, 15); // 从第2个Byte位置开始，放置一个数字为15的Short类型数据(占4 Byte)

dataView.setInt32(6, long.high); // 采用大端字节序放置
dataView.setInt32(10, long.low);

// 数据读取过程

let shortNumber = dataView.getInt16(0);
let intNumber = dataView.getInt32(2);

let longNumber = Long.fromBits(dataView.getInt32(10), dataView.getInt32(6)); // 根据大端字节序读取，该构造函数入参依次为：低16位，高16位
```

通过上面的示例，我们将一串二进制数据转换成为了JavaScript中通用的数据类型。

# 总结

通过使用ArrayBuffer和DataView，我们能够快速的将数字数据从二进制转换为JavaScript常用数据类型如Int、Short等；同时，我们也可以将这些数据类型转换为二进制数据。有了这些基础知识，我们就能够理解在之后的博客中讲到的关于使用WebSocket进行二进制数据传递的过程和处理逻辑。

下一篇博客我们将介绍String类型相关的二进制处理与转换操作，有兴趣的同学可以关注留意下相关内容。

# 部分参考资料

[阮一峰老师关于字节序的介绍](http://www.ruanyifeng.com/blog/2016/11/byte-order.html)
