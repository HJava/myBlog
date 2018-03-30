# 概述

本文是WebSocket系列的第二篇，主要是通过介绍如何在WebSocket中使用ArrayBuffer来进行数据的发送，同时解决接收到二进制数据后如何转换为JavaScript中常见的数据类型的问题。

本文需要读者对WebSocket基础知识有一定的了解，如果读者并没有相关的知识储备，可以阅读WebSocket系列的第一篇——[WebSocket系列之基础知识入门篇](https://juejin.im/post/5ab91ac96fb9a028db58b1d5)。

本文主要内容有：

- 为什么在WebSocket传输过程中使用二进制数据。


- JavaScript数据类型如何与强类型语言数据类型对应。
- 如何将JavaScript数据类型转换为二进制数据（以Long型数据和String数据为例）


- 如何将WebSocket收到的二进制数据转换为JavaScript数据类型（以Long型数据和String数据为例）

虽然本文只针对Long型数据和String型数据进行了具体说明，但是其他的数据类型如Int，Boolean等处理方法也完全相同，大家只要按照相同的方法就能够解决。

# 为什么在WebSocket传输过程中使用二进制数据

WebSocket能够传递`String`、`ArrayBuffer`和`Blob`三种数据类型。`Blob`作为一个类文件对象，我们暂且不讨论。`String`和`ArrayBuffer`在数据传递上都能够满足我们的需求，比如`String`可以使用通用的JSON格式，`ArrayBuffer`能够使用二进制数据格式。

那么，`ArrayBuffer`相较于`String`到底有什么优势呢，我认为有一下几点：

- 同等内容下传输长度较短。在表示同等数据量情况下，`ArrayBuffer`表示的二进制数据大小明显小于使用JSON格式的`String`。以`Boolean`类型为例，在二进制中，表示`Boolean`类型最少只需要1个Bit来标识（在JavaScript中为了便于处理，一般用一个Byte来表示，1 Byte = 8 Bit）；而在JSON中，则需要4个字母来表示`true`（WebSocket传输数据时使用的是UTF-16字符串，是个变长编码），存储长度很明显较长。
- 具有一定的安全性。二进制编码相较于明文的JSON编码具有一定的安全性。这个优势是相较于JSON不进行编码明文传输而言的，在别人获取你的编码方式后，二进制传输方式也是不安全的。
- 具有良好的兼容性。只要处理方式一致，二进制数据不会有编码相关的兼容问题，而使用WebSocket进行String类型传输时，有可能需要其他客户端进行UTF-16编码转换到UTF-8编码的过程。

基于这些优点，当前的项目在线上环境也使用了二进制数据的方案。

# JavaScript数据类型如何与强类型语言数据类型对应

说完了使用二进制数据来进行传输的优点，我们来看下强类型中的一些数据类型在JavaScript该如何表示。

在强类型语言中，有一些通用的数据类型，在JavaScript中是不存在的，例如Long，Short等。但是，我们可以通过JavaScript中已有的数据类型，来表示这些没有的数据类型，具体表示方法如下：

| 强类型语言类型 | JavaScript语言类型 | 备注                                                         |
| -------------- | ------------------ | ------------------------------------------------------------ |
| Boolean        | Boolean            |                                                              |
| Byte           | Number             |                                                              |
| Short          | Number             | Number型的范围包含了Short型                                  |
| Int            | Number             | Number型的范围包含了Int型                                    |
| Long           | Long型对象         | 具体实现方式可以见我写过的博客——[Long.js源码分析与学习](https://juejin.im/post/5a88e148f265da4e761fd400) |
| String         | String             |                                                              |

通过上面的数据类型对应，我们可以将JavaScript中不存在的数据类型转换为已有的数据类型来表示。我们只需要在转换的过程中注意相关类型的边界（多数情况出现在JavaScript类型转换为强语言类型的时候）问题。

# 如何将JavaScript数据类型转换为二进制数据

在了解了相关的数据类型转换对应关系之后，我们来讲讲怎么把JavaScript中的数据类型转换成为二进制数据。

# 如何将WebSocket收到的二进制数据转换为JavaScript数据类型