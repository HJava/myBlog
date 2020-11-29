# 概述
通过前三篇博客，我们能够了解在通过WebSocket发送数据之前，我们需要传递的数据是如何变成ArrayBuffer二进制数据的；在我们收到二进制数据之后，我们又如何将其变成了JavaScript中的常见数据类型。
本文作为WebSocket系列的第四篇内容，将会用一个简单的IM聊天应用把整个WebSocket传输二进制数据类型的内容连接起来，让用户对整个WebSocket传输二进制数据的方法有个了解。
本文的主要内容如下：
- 如何设计一个二进制协议
- WebSocket如何发送二进制数据
- WebSocket如何处理接收的二进制数据
之前的博客我们介绍过了WebSocket基础知识，数字类型和字符串类型与二进制数据间的转换，如果没有相关的基础，建议先依次阅读以下文章：
- [WebSocket系列之基础知识入门篇](https://juejin.im/post/5ab91ac96fb9a028db58b1d5)
- [WebSocket系列之JavaScript中数字数据如何转换为二进制数据](https://juejin.im/post/5abb560a6fb9a028d141262b)
- [WebSocket系列之字符串如何与二进制数据进行转换](https://juejin.im/post/5abdc38ef265da2375070008)
# 如何设计一个二进制协议
## 什么是协议
> 协议，网络协议的简称，网络协议是通信计算机双方必须共同遵从的一组约定。如怎么样建立连接、怎么样互相识别等。只有遵守这个约定，计算机之间才能相互通信交流。它的三要素是：语法、语义、时序。  

通过[百度百科](https://baike.baidu.com/item/%E5%8D%8F%E8%AE%AE/13020269)中的介绍，我们对协议的概念有了一个基础的了解。通俗来说，协议就是通信双方约定好的一套规则。
## 为什么要设计协议
没有规矩不成方圆。通信双方只有通过协议，才能够识别对方发送的数据内容。
## 我们应该如何设计这套协议
首先，协议的设计应该能够区分不同的各个数据包；其次，它还需要具备一定的兼容性。
根据上述两点要求，我们设计了一套简单的IM聊天协议，支持文本、图片、文件三种消息。这套协议是按照最简单的思路来设计的，因此也只是给大家一个参考的观点，在真正的线上使用场景中，协议会比本文中的复杂和更加有层次。具体格式如下：
```json
{
    "id": "short", // 消息类型，1是文本协议格式；2是图片协议格式；3是文件协议格式
    "sender": "long", // 发送用户唯一id
    "reciever": "long", // 接受用户唯一id
    "data": "string" // 消息内容，如果是文本协议则为文本内容；如果是图片协议则为图片地址；如果是文件协议则为文件地址
}
```
## 这套协议如何使用
### 发送消息
从协议格式可知，将上述数据按照上述固定顺序放入ArrayBuffer中，即可得到一个有特定含义的二进制数据。例如：
```json
{
    "id": 1,
    "sender": "123",
    "reciever": "456",
    "data": "Hellow World"
}
```
当我们需要发送此消息时，只需要：
1. 在前2个Byte放入`id`。
2. 接下来8个Byte中放入`sender`。
3. 再接下来8个Byte放入`reciever`。
4. 最后紧接着放入一个string类型（以[WebSocket系列之字符串如何与二进制数据进行转换](https://juejin.im/post/5abdc38ef265da2375070008)博客中的格式为例，先将字符串长度构造成一个int类型，放在前4个Byte中，接下来将string类型编码后放入）。
此数据就完全按照协议构造完成了。我们只需将次协议通过WebSocket发送即可。具体方法将会在后面章节中说明。
### 接收消息
从协议格式可知，当我们收到一条消息时，只需要按照协议规范来进行反向解析即可。例如：
```json
{
    "id": 1,
    "sender": "123",
    "reciever": "456",
    "data": "Hellow World"
}
```
如果发送端发送的数据仍然为此消息，我们的解析顺序为：
1. 先根据前2个Byte读取一个Short类型的`id`数值。
2. 将接下来的8个Byte读取为Long类型的`sender`字段。
3. 再接下来的8个Byte读取为Long类型的`reciever`字段。
4. 接下来读取一个string类型（以`发送消息`这一节的数据为例，先读取4个Byte长度的int类型字符串长度，然后再根据长度读取字符串即可）。
### 扩展此协议
当此协议字段无法满足并且已经在线上使用时，我们应该如何扩展呢？
根据我们的写入和读取步骤，我们可以知道：**每次我们读取的二进制数据可以认为是一个格式固定的数据（string类型在构造时会有长度信息，因此认为也是长度相对固定），所以我们在读取二进制数据时读取的长度也是固定的。**因此，我们在扩展时只需要往协议后面增加字段即可。
- 扩展前的应用仍然会读取之前已经确定的数据协议，只需要保证内容不变，那么功能也不会变。
- 扩展后的应用能够解析扩展后的协议，因此得到更多的数据，从而可以实现更多的功能。
# WebSocket如何发送二进制数据
通过`如何设计一个二进制协议`一章，我们知道了如何定义WebSocket传输的二进制数据格式。下面，我们来看下如何在WebSocket中发送二进制数据：
```javascript
let arrayBuffer = getArrayBufferMessagesFromUser(); // 获取用户需要发送的消息数据,为一个ArrayBuffer对象
let webSocket = getWebSocket(); // 获取已经连接成功的WebSocket实例

websocket.binaryType = 'arraybuffer'; // 指定WebSocket接受ArrayBuffer实例作为参数

webSocket.send(arrayBuffer);
```
通过上面的示例我们可以知道，WebSocket在发送string类型的数据或者ArrayBuffer类型的数据时，使用的API接口都是`send`方法，我们只需要在WebSocket初始化后指定传输类型`binaryType`即可。
# WebSocket如何处理接收的二进制数据
通过`WebSocket如何发送二进制数据`一章，我们知道了如何发送二进制数据。接下来，让我们开看下如何处理WebSocket接收到的二进制数据：
```
let webSocket = getWebSocket(); // 获取已经连接成功的WebSocket实例

websocket.binaryType = 'arraybuffer'; // 指定WebSocket接受ArrayBuffer实例作为参数

webSocket.addEventListener('message', (message) => {
    let arrayBuffer = message.data; // 获取用户接收到的消息数据,为一个ArrayBuffer对象
    let data = parseMessage(arrayBuffer); // 解析二进制数据
});
```
通过上面的示例我们可以知道，当我们在建立连接时指定了传输类型`binaryType`为ArrayBuffer之后，我们通过WebSocket收到的数据也是一个ArrayBuffer实例。我们只需要根据第一章讲解的方式进行解析即可。
# 总结
本文作为WebSocket系列的第四篇，通过一个IM聊天应用的示例将前三篇博客分享的内容串联起来，给读者完整介绍了在WebSocket中使用二进制数据进行传输的方法以及相关的数据类型转换。
通过前面4篇博客的内容，读者可以根据自己的需求快速的构造和封装WebSocket进行二进制数据传输，基本能够串联整个应用流程。
WebSocket系列下一篇文章将会介绍在线上环境中，如何保证WebSocket的连接，以及线上可能遇到的问题。通过应对WebSocket可能出现的问题，我们能够让整个长连接更加健壮。