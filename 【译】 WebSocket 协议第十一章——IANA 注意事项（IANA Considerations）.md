
## 概述

本文为 WebSocket 协议的第十一章，本文翻译的主要内容为 WebSocket 的 IANA 相关注意事项。

有兴趣了解该文档之前几章内容的同学可以见：

- [【译】WebSocket 协议——摘要（ Abstract ）][1]
- [【译】WebSocket 协议第一章——介绍（ Introduction ）][2]
- [【译】WebSocket 协议第二章——一致性要求（ Conformance Requirements ）][3]
- [【译】WebSocket 协议第三章——WebSocket网址（ WebSocket URIs ）][4]
- [【译】WebSocket 协议第四章——连接握手（ Opening Handshake ）][5]
- [【译】WebSocket 协议第五章——数据帧（Data Framing）][6]
- [【译】WebSocket 协议第六章——发送与接收消息（Sending and Receiving Data）][7]
- [【译】 WebSocket 协议第七章——关闭连接（Closing the Connection）][8]
- [【译】 WebSocket 协议第八章——错误处理（Error Handling）][9]
- [【译】 WebSocket 协议第九章——扩展（Extension）][10]
- [【译】 WebSocket 协议第十章——安全性考虑（Security Considerations）][11]

## IANA 注意事项（协议正文）

### 11.1 注册新 URI 协议

#### 11.1.1 注册 “wx” 协议

`ws` URI 定义了 WebSocket 服务器和资源名称。

URI 协议名称

ws

状态

永久

URI 协议语法

使用 ABNF （[RFC5234][12]）语法和来自 URI 规范 [RFC3986][13] 的  ABNF 终端：

"ws:" "//" authority path-abempty [ "?" query ]

`path-abempty` 和 `query`  [RFC3986][14] 部分组成了发送给服务端的资源名称，来标记需要的服务类型。其他的部分在 [RFC3986][15] 中定义了含义。

URI 语义方案

这个方案的唯一操作就是使用 WebSocket 协议打开一个连接。

编码注意事项

按照上面定义的语法排除的主机部分中的字符必须按照 [RFC3987][16] 中的规定从 Unicode 转换为 ASCII 或其替换字符。为了实现基于方案的规范化，国际化网域名称（IDN）




[1]:	https://juejin.im/post/5b12966fe51d450689495e41
[2]:	https://juejin.im/post/5b1a7189e51d45068b496cf0
[3]:	https://juejin.im/post/5b1e6beae51d4506b62cbd64
[4]:	https://juejin.im/post/5b226d716fb9a00e594c5da5
[5]:	https://juejin.im/post/5b2b9850518825748e545d23
[6]:	https://juejin.im/post/5c32f906f265da6136229fac
[7]:	https://juejin.im/post/5c33648b6fb9a049b2220bb7
[8]:	https://juejin.im/post/5c3c3982f265da613f2fafe4
[9]:	https://juejin.im/post/5c4c1d8cf265da61285a759d
[10]:	https://juejin.im/post/5c4c8b82f265da616a4801ca
[11]:	https://juejin.im/post/5c556541f265da2da23d019d
[12]:	https://tools.ietf.org/html/rfc5234
[13]:	https://tools.ietf.org/html/rfc3986
[14]:	https://tools.ietf.org/html/rfc3986
[15]:	https://tools.ietf.org/html/rfc3986
[16]:	https://tools.ietf.org/html/rfc3987