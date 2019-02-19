
## 概述

本文为 WebSocket 协议的第十二章，本文翻译的主要内容为如何使用其他规范中的 WebSocket 协议。

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
- [【译】 WebSocket 协议第十一章——IANA 注意事项（IANA Considerations）][12]

## 使用其他规范中的WebSocket协议（协议正文）

WebSocket协议旨在由另一规范使用，以提供动态作者定义内容的通用机制。例如，在定义脚本 API 的规范中定义 WebSocket 协议。

例如一个规范首先需要`建立 WebSocket 连接`，提供该算法：

- 目标资源，包含一个`主机名（host）`和一个`端口（port）`。
- `资源名称`，允许在一个主机和端口上识别多个服务。
- `安全`标记，当这个值为 true 时，连接应该被加密，如果为 false 时则不需要。
- 原始[RFC6454][13]的ASCII序列化，负责连接。
- 可选的，基于 WebSocket 连接的通过一个字符串定义的协议。

`主机`、`端口`、`资源名称`和`安全`标记通常是使用解析 WebSocket URI 组件，通过 URI 来获取。如果 URI 中没有指定这些 WebSocket 字段，那么这个解析将失败。

如果在任意时间连接被关闭了，那么规范需要使用`关闭 WebSocket 连接`算法（第 7.1.1 节）。

第 7.1.4 节定义了什么时候`WebSocket 连接关闭`。

当连接打开时，文档需要处理`收到一条 WebSocket 消息`（第 6.2 节）的场景。

为了向已经建立的连接发送一些`数据`，文档需要处理`发送 WebSocket 消息`（第 6.1 节）。

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
[12]:	https://juejin.im/post/5c6a9b8f518825629a77b788
[13]:	https://tools.ietf.org/html/rfc6454