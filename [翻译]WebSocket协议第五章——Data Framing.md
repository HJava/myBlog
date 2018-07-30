# 概述

本文为WebSocket协议的第五章，本文翻译的主要内容为WebSocket传输的数据相关内容。

# 数据帧（协议正文）

## 5.1 概览

在WebSocket协议中，数据是通过一系列数据帧来进行传输的。为了避免由于网络中介（例如一些拦截代理）或者一些在第10.3节讨论的安全原因，客户端必须在它发送到服务器的所有帧中添加掩码（Mask）（具体细节见5.3节）。（注意：无论WebSocket协议是否使用了TLS，帧都需要添加掩码）。服务端收到没有添加掩码的数据帧以后，必须立即关闭连接。在这种情况下，服务端可以发送一个在7.4.1节定义的状态码为1002（协议错误）的关闭帧。服务端禁止在发送数据帧给客户端时添加掩码。客户端如果收到了一个添加了掩码的帧，必须立即关闭连接。在这种情况下，它可以使用第7.4.1节定义的1002（协议错误）状态码。（这些规则可能会在将来的规范中放开）。

基础的数据帧协议使用操作码、有效负载长度和在“有效负载数据”中定义的放置“扩展数据”与“引用数据”的指定位置来定义帧类型。特定的bit位和操作码为将来的协议扩展做了保留。

一个数据帧可以在开始握手完成之后和终端发送了一个关闭帧之前的任意一个时间通过客户端或者服务端进行传输（第5.5.1节）。

## 5.2 基础帧协议

在这节中的这种数据传输部分的有线格式是通过ABNF[RFC5234](https://tools.ietf.org/html/rfc5234)来进行详细说明的。(注意：不像这篇文档中的其他章节内容，在这节中的ABNF是对bit组进行操作。每一个bit组的长度是在评论中展示的。在线上编码时，最高位的bit是在ABNF最左边的)。对于数据帧的高级的预览可以见下图。如果下图指定的内容和这一节中后面的ABNF指定的内容有冲突的话，以下图为准。

```
      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-------+-+-------------+-------------------------------+
     |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
     |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
     |N|V|V|V|       |S|             |   (if payload len==126/127)   |
     | |1|2|3|       |K|             |                               |
     +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
     |     Extended payload length continued, if payload len == 127  |
     + - - - - - - - - - - - - - - - +-------------------------------+
     |                               |Masking-key, if MASK set to 1  |
     +-------------------------------+-------------------------------+
     | Masking-key (continued)       |          Payload Data         |
     +-------------------------------- - - - - - - - - - - - - - - - +
     :                     Payload Data continued ...                :
     + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
     |                     Payload Data continued ...                |
     +---------------------------------------------------------------+
```

FIN: 1 bit

​	表示这是消息的最后一个片段。第一个片段也有可能是最后一个片段。

RSV1，RSV2，RSV3: 每个1 bit

​	必须设置为0，除非扩展了非0值含义的扩展。如果收到了一个非0值但是没有扩展任何非0值的含义，接收终端必须断开WebSocket连接。

Opcode: 4 bit

​	定义“有效负载数据”的解释。如果收到一个未知的操作码，接收终端必须断开WebSocket连接。下面的值是被定义过的。

​	%x0 表示一个持续帧

​	%x1 表示一个文本帧

​	%x2 表示一个二进制帧

​	%x3-7 预留给以后的非控制帧

​	%x8 表示一个连接关闭包

​	%x9 表示一个ping包

​	%xA 表示一个pong包

​	%xB-F 预留给以后的控制帧

Mask: 1 bit

​	mask标志位，定义“有效负载数据”是否添加掩码。如果设置为1，那么掩码的键值存在于Masking-Key中，根据5.3节描述，这个一般用于解码“有效负载数据”。所有的从客户端发送到服务端的帧都需要设置这个bit位为1。

Payload length: 7 bits, 7+16 bits, or 7+64 bits

​	以字节为单位的“有效负载数据”长度，如果值为0-125，那么就表示负载数据的长度。如果是126，那么接下来的2个bytes解释为16bit的无符号整形作为负载数据的长度。如果是127，那么接下来的8个bytes解释为一个64bit的无符号整形（最高位的bit必须为0）作为负载数据的长度。多字节长度量以网络字节顺序表示（译注：应该是指大端序和小端序）。在所有的示例中，长度值必须使用最小字节数来进行编码，例如：长度为124字节的字符串不可用使用序列126,0,124进行编码。有效负载长度是指“扩展数据”+“应用数据”的长度。“扩展数据”的长度可能为0，那么有效负载长度就是“应用数据”的长度。

Masking-Key: 0 or 4 bytes

​	所有从客户端发往服务端的数据帧都已经与一个包含在这一帧中的32 bit的掩码进行过了运算。如果mask标志位（1 bit）为1，那么这个字段存在，如果标志位为0，那么这个字段不存在。在5.3节中会介绍更多关于客户端到服务端增加掩码的信息。

Payload data: (x+y) bytes

​	“有效负载数据”是指“扩展数据”和“应用数据”。

Extension data: x bytes

​	除非协商过扩展，否则“扩展数据”长度为0 bytes。在握手协议中，任何扩展都必须指定“扩展数据”的长度，这个长度如何进行计算，以及这个扩展如何使用。如果存在扩展，那么这个“扩展数据”包含在总的有效负载长度中。

Application data: y bytes

​	任意的“应用数据”，占用“扩展数据”后面的剩余所有字段。“应用数据”的长度等于有效负载长度减去“扩展应用”长度。



基础数据帧协议通过ABNF进行了正式的定义。需要重点知道的是，这些数据都是二进制的，而不是ASCII字符。例如，长度为1 bit的字段的值为%x0 / %x1代表的是一个值为0/1的单独的bit，而不是一整个字节（8 bit）来代表ASCII编码的字符“0”和“1”。一个长度为4 bit的范围是%x0-F的字段值代表的是4个bit，而不是字节（8 bit）对应的ASCII码的值。不要指定字符编码：“规则解析为一组最终的值，有时候是字符。在ABNF中，字符仅仅是一个非负的数字。在特定的上下文中，会根据特定的值的映射（编码）编码集（例如ASCII）”。在这里，指定的编码类型是将每个字段编码为特定的bits数组的二进制编码的最终数据。



ws-frame =

- frame-fin; 长度为1 bit
- frame-rsv1; 长度为1 bit
- frame-rsv2; 长度为1 bit
- frame-rsv3; 长度为1 bit
- frame-opcode; 长度为4 bit
- frame-masked; 长度为1 bit
- frame-payload-length; 长度为7或者7+16或者7+64 bit
- [frame-masking-key]; 长度为32 bit
- frame-payload-data; 长度为大于0的n*8 bit（其中n>0）

frame-fin =

- %x0，除了以下为1的情况
- %x1，最后一个消息帧
- 长度为1 bit

frame-rsv1 =

- %x0 / %x1，长度为1 bit，如果没有协商则必须为0

frame-rsv2 =

- %x0 / %x1，长度为1 bit，如果没有协商则必须为0

frame-rsv3 =

- %x0 / %x1，长度为1 bit，如果没有协商则必须为0

frame-opcode =

- frame-opcode-non-control
- frame-opcode-control
- frame-opcode-cont

frame-opcode-non-control

- %x1，文本帧
- %x2，二进制帧
- %x3-7，保留给将来的非控制帧
- 长度为4 bit

frame-opcode-control

- %x8，连接关闭
- %x9，ping帧
- %xA，pong帧
- %xB-F，保留给将来的控制帧
- 长度为4 bit

frame-masked

- %x0，不添加掩码，没有frame-masking-key
- %x1，添加掩码，存在frame-masking-key
- 长度为1 bit

frame-payload-length

- %x00-7D，长度为7 bit
- %x7E frame-payload-length-16，长度为7+16 bit
- %x7F frame-payload-length-63，长度为7+64 bit

frame-payload-length-16

- %x0000-FFFF，长度为16 bit

frame-payload-length-63

- %x0000000000000000-7FFFFFFFFFFFFFFF，长度为64 bit

frame-masking-key

- 4(%x00-FF)，当frame-mask为1时存在，长度为32 bit

frame-payload-data

- frame-masked-extension-data frame-masked-application-data，当frame-masked为1时
- frame-unmasked-extension-data frame-unmasked-application-data，当frame-masked为0时

frame-masked-extension-data

- \*(%x00-FF)，保留给将来的扩展，长度为n\*8，其中n>0

frame-masked-application-data

- \*(%x00-FF)，长度为n\*8，其中n>0

frame-unmasked-extension-data

- \*(%x00-FF)，保留给将来的扩展，长度为n\*8，其中n>0

frame-unmasked-application-data

- \*(%x00-FF)，长度为n\*8，其中n>0

## 5.3 客户端到服务端添加掩码 

添加掩码的数据帧必须像5.2节定义的一样，设置frame-masked字段为1。

掩码值像第5.2节说到的完全包含在帧中的frame-masking-key上。

# 总结