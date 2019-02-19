
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

URI 协议含义

这个方案的唯一操作就是使用 WebSocket 协议打开一个连接。

编码注意事项

按照上面定义的语法排除的主机部分中的字符必须按照 [RFC3987][16] 中的规定从 Unicode 转换为 ASCII 或其替换字符。为了实现基于方案的规范化，国际化网域名称（IDN）主机组件的形式与 punycode 码之间的相互转化认为是等价的（见 [RFC3987 第 5.3.3 节][17]）。

除了由上面语法排除的字符外，其他组件的字符在第一次转换为 UTF-8 字符时，必须从 Unicode 码转化到 ASCII 码，然后使用百分比编码格式替换对应的定义在 URI [RFC3896][18] 字符和国际化资源标识符（IRI） [RFC3987][19] 规范。

应用/协议使用这个 URI 协议规范

WebSocket Protocol

互操作性注意事项

使用 WebSocket 时需要使用 1.1 或者更高版本的 HTTP 协议。

安全性注意事项

见”安全性注意事项”一节。

联系

HYBI WG \<hybi@ietf.org\\\\\\\>

作者/更改控制者

IETF \<iesg@ietf.org\\\\\\\>

关联

[RFC 6455][20]

#### 11.1.2 注册 “wss”协议

一个 `wss` 的 URI 定义了一个 WebSocket 服务器和资源名称，表明通过这个链接的传输需要通过 TLS（包含标准的 TLS 能力例如数据保密性和完整性以及终端认证）来进行保护。

URI 协议名称

wss

状态

永久

URI 协议语法

使用 ABNF （[RFC5234][21]）语法和来自 URI 规范 [RFC3986][22] 的  ABNF 终端：

"wss:" "//" authority path-abempty [ "?" query ]

`path-abempty` 和 `query`  [RFC3986][23] 部分组成了发送给服务端的资源名称，来标记需要的服务类型。其他的部分在 [RFC3986][24] 中定义了含义。

URI 协议含义

这个方案的唯一操作就是使用 WebSocket 协议打开一个连接，通过 TLS 加密。

编码注意事项

按照上面定义的语法排除的主机部分中的字符必须按照 [RFC3987][25] 中的规定从 Unicode 转换为 ASCII 或其替换字符。为了实现基于方案的规范化，国际化网域名称（IDN）主机组件的形式与 punycode 码之间的相互转化认为是等价的（见 [RFC3987 第 5.3.3 节][26]）。

除了由上面语法排除的字符外，其他组件的字符在第一次转换为 UTF-8 字符时，必须从 Unicode 码转化到 ASCII 码，然后使用百分比编码格式替换对应的定义在 URI [RFC3896][27] 字符和国际化资源标识符（IRI） [RFC3987][28] 规范。

应用/协议使用这个 URI 协议规范

WebSocket Protocol

互操作性注意事项

使用 WebSocket 时需要使用 1.1 或者更高版本的 HTTP 协议。

安全性注意事项

见”安全性注意事项”一节。

联系

HYBI WG \<hybi@ietf.org\\\\\\\>

作者/更改控制者

IETF \<iesg@ietf.org\\\\\\\>

关联

[RFC 6455][29]


### 11.2 注册“WebSocket”协议升级关键值

这一节描述一个在 HTTP 升级凭证注册的关键值，在 [RFC2817][30] 定义。

凭证名称

WebSocket

作者/更改控制者

IETF \<iesg@ietf.org\\\\\\\>

关联

[RFC 6455][31]

### 11.3 注册新的 HTTP 头字段

#### Sec-WebSocket-Key

这一节描述一个注册在永久消息头字段名称中的头字段，在 [RFC3864][32] 定义。

头字段名称

Sec-WebSocket-Key

应用协议

http

状态

标准

作者/更改控制者

IETF

说明文档

[RFC 6455][33]

关联信息

这个头字段只用于 WebSocket 开始握手。

`Sec-WebSocket-Key` 头字段是用在 WebSocket 开始握手阶段。它是通过客户端发送给服务端，这部分信息用于服务端证明收到一个有效的 WebSocket 握手操作的认证。这可以帮助确认服务端不会接收可能被用来向 WebSocket 服务任意发送数据的非 WebSocket 客户端的连接（例如 HTTP 客户端）。

`Sec-WebSocket-Key` 头字段禁止在一个 HTTP 请求中出现多次。

#### 11.3.2 Sec-WebSocket-Extensions

这一节描述一个注册在永久消息头字段名称中的头字段，在 [RFC3864][34] 定义。

头字段名称

Sec-WebSocket-Extensions

应用协议

http

状态

标准

作者/更改控制者

IETF

说明文档

[RFC 6455][35]

关联信息

这个头字段只用于 WebSocket 开始握手。

`Sec-WebSocket-Extensions` 头字段是用于 WebSocket 开始握手阶段。它最开始是通过客户端发送给服务端，然后通过服务端发送给客户端，来对一个在连接中的协议级的扩展进行协商。

`Sec-WebSocket-Extensions` 头字段可能会在一个 HTTP 请求中出现多次（这个逻辑是等价于一个单独的 `Sec-WebSocket-Extensions` 头字段包含所有值）。然而，`Sec-WebSocket-Extensions` 头字段在 HTTP 响应中不能出现超过1次。

#### 11.3.3 Sec-WebSocket-Accept

这一节描述一个注册在永久消息头字段名称中的头字段，在 [RFC3864][36] 定义。

头字段名称

Sec-WebSocket-Accept

应用协议

http

状态

标准

作者/更改控制者

IETF

说明文档

[RFC 6455][37]

关联信息

这个头字段只用于 WebSocket 开始握手。

`Sec-WebSocket-Accpet` 头字段是用于 WebSocket 开始握手阶段。它是通过服务端发送给客户端，用来确认服务端会初始化一个 WebSocket 连接。

`Sec-WebSocket-Accpet` 头在一个 HTTP 响应中不允许出现超过1次。

#### 11.3.4 Sec-WebSocket-Protocol

这一节描述一个注册在永久消息头字段名称中的头字段，在 [RFC3864][38] 定义。

头字段名称

Sec-WebSocket-Protocol

应用协议

http

状态

标准

作者/更改控制者

IETF

说明文档

[RFC 6455][39]

关联信息

这个头字段只用于 WebSocket 开始握手。

`Sec-WebSocket-Protocol` 头字段是用于 WebSocket 开始握手阶段。它是从客户端发送给服务端，然后从服务端返回给服务端来确认连接的子协议。这个机制能够让双方选择一个子协议，同时向服务端确认可以支持这个子协议。

`Sec-WebSocket-Protocol` 头字段可以在一个 HTTP 请求中出现多次（这个逻辑是等价于一个单独的 `Sec-WebSocket-Protocol` 头字段包含所有值）。然而，￼ `Sec-WebSocket-Protocol` 头字段在 HTTP 响应中不能出现超过1次。

#### 11.3.5 Sec-WebSocket-Version

这一节描述一个注册在永久消息头字段名称中的头字段，在 [RFC3864][40] 定义。

头字段名称

Sec-WebSocket-Version

应用协议

http

状态

标准

作者/更改控制者

IETF

说明文档

[RFC 6455][41]

关联信息

这个头字段只用于 WebSocket 开始握手。

`Sec-WebSocket-Version` 头字段是用于 WebSocket 开始握手阶段。它是从客户端发送给服务端来表示这个连接使用的协议版本。它能够让服务端正确的进行开始握手和接下来的数据发送，以及在服务端不能够在一个安全方式下正确解析数据时关闭连接。`Sec-WebSocket-Version` 头字段在服务端理解的版本不匹配从客户端收到的版本导致的 WebSocket 握手失败时，也从服务端发送给客户端。在这种情况下，这个头字段包含服务端支持的协议版本。

注意这里不期望更高的版本号需要向前兼容低版本号。

`Sec-WebSocket-Version` 头字段可以在一个 HTTP 响应中出现多次（这个逻辑等价于一个单独的`Sec-WebSocket-Version`包含所有的值）。然而，`Sec-WebSocket-Version` 头字段不能在 HTTP 请求中出现超过1次。

### 11.4 WebSocket 扩展名注册表

这个规范根据[RFC5526][42]中规定的原则为 WebSocket 协议创建了一个新的 IANA 注册表，用于 WebSocket 扩展名称。

作为此注册表的一部分，IANA 包含了一下信息：

扩展定义

这个扩展的定义，将在 `Sec-WebSocket-Extensions` 头字段中使用，在此规范的第 11.3.2 节注册。这个值必须满足在此规范第 9.1 节中定义的扩展凭证要求。

扩展通用名

扩展名称，一般称为扩展名。

扩展定义

对定义与 WebSocket 协议一起使用的扩展的文档的引用。

已知不兼容扩展

已知的不兼容的扩展定义列表。

WebSocket 扩展名是受到“先到先得” IANA 注册政策 [RFC5226][43] 限制的。

这个注册表里没有初始值。

### 11.5 WebSocket 子协议名注册表

这个规范根据[RFC5526][44]中规定的原则为 WebSocket 协议创建了一个新的 IANA 注册表，用于 WebSocket 扩展名称。

作为此注册表的一部分，IANA 包含了一下信息：

子协议定义

子协议的标识符将在 `Sec-WebSocket-Protocol` 头字段中使用，在此规范的第 11.3.4 节中注册。这个之必须符合此规范第 4.1 节中的第 10  项要求—换句话说，这个之必须是 [RFC2616][45] 中定义的凭证。

子协议通用名

子协议名称，通常被称为子协议。

子协议定义

对定义与 WebSocket 协议一起使用的子协议的文档的引用。

WebSocket 子协议名是受到“先到先得” IANA 注册政策 [RFC5226][46] 限制的。

### 11.6 WebSocket 版本号注册表

该规范根据 [RFC5226][47] 中规定的原则为 WebSocket 协议创建了一个新的 IANA 注册表，用于 WebSocket 版本号。

作为此注册表的一部分，IANA 包含了一下信息：

版本号

版本号是用于在此规范第 4.1 节中制定的 `Sec-WebSocket-Version` 字段。这个值必须是一个范围在 0 到 255（含）之间的非负整数。

参考

RFC 请求新版本号或者带有版本号的草稿名称（见下文）。

状态

临时或者标准。见下面描述。

版本号被指定为“临时”或者“标准”。

“标准”的版本号被记录在 RFC 文档中，被认为是一个重大、稳定的 WebSocket 协议版本，例如定义在这个 RFC 中的版本。“标准”版本号是受到 “IETF 评论” IANA 注册政策 [RFC5526][48] 限制的。

“临时”版本是记录在网络草案和用于帮助实现者识别 WebSocket 协议的已部署版本并与之互操作，例如开发后但是发布前的 RFC 版本。“临时”版本号是受到 “专家评论” IANA 注册政策 [RFC5526][49] 、 最初的指定专家如HYBI 工作组主席（或者，如果工作组关闭，那么是 IETF 应用领域的领域主任）限制的。

IANA 已经向注册表中添加了如下的初始值。

| 版本号 | 引用                                                         | 状态 |
| ------ | ------------------------------------------------------------ | ---- |
| 0      | [draft-ietf-hybi-thewebsocketprotocol-00][50] | 临时 |
| 1      | [draft-ietf-hybi-thewebsocketprotocol-01][51] | 临时 |
| 2      | [draft-ietf-hybi-thewebsocketprotocol-02][52] | 临时 |
| 3      | [draft-ietf-hybi-thewebsocketprotocol-03][53] | 临时 |
| 4      | [draft-ietf-hybi-thewebsocketprotocol-04][54] | 临时 |
| 5      | [draft-ietf-hybi-thewebsocketprotocol-05][55] | 临时 |
| 6      | [draft-ietf-hybi-thewebsocketprotocol-06][56] | 临时 |
| 7      | [draft-ietf-hybi-thewebsocketprotocol-07][57] | 临时 |
| 8      | [draft-ietf-hybi-thewebsocketprotocol-08][58] | 临时 |
| 9      | 保留                                                         |      |
| 10     | 保留                                                         |      |
| 11     | 保留                                                         |      |
| 12     | 保留                                                         |      |
| 13     | [RFC6455][59]               |      |

### 11.7 WebSocket 关闭码注册表

该规范根据 [RFC5226][60] 中规定的原则为 WebSocket 协议创建了一个新的 IANA 注册表，用于 WebSocket 关闭码。

作为此注册表的一部分，IANA 包含了一下信息：

状态码

状态码表示定义在此文档第 7.4 节中 WebSocket 连接关闭的原因。这个状态码是一个在 1000 到 4999（含）之间的整数。

含义

状态码含义。每一个状态码有一个特定的含义。

联系

保留状态代码的实体的联系人。

关联

请求状态码的固定文档和含义定义。对于 1000-2999 的状态码来说是必须的，推荐使用 3000-3999 范围的状态码。

WebSocket 关闭状态码根据它的范围有不同的注册要求。使用在这个协议和它的后续的版本或者扩展的请求版本号是受到“标准行为”、“规范要求”（这意味着“指定专家”）或者“IESG 评论” IANA注册表政策限制的，应该在 1000-2999 范围内授权。被类库、框架和应用使用的状态码是受限制于“先到先得”IANA 注册表政策，应该在 3000-3999 范围内授权。4000-4999 范围的状态码是私用的。请求应指明它们是否正在通过扩展、类库、框架或者应用使用请求WebSocket协议的状态代码（或者将来的协议的版本）。

IANA已经向注册表中添加了如下初始值。



| 状态码 | 含义             | 联系人        | 关联                                       |
| ------ | ---------------- | ------------- | ---------------------------------------------- |
| 1000   | 正常关闭         | hybi@ietf.org | [RFC6455][61] |
| 1001   | 离开             | hybi@ietf.org | [RFC6455][62] |
| 1002   | 协议错误         | hybi@ietf.org | [RFC6455][63] |
| 1003   | 不支持的数据类型 | hybi@ietf.org | [RFC6455][64] |
| 1004   | 保留             | hybi@ietf.org | [RFC6455][65] |
| 1005   | 没有收到状态码   | hybi@ietf.org | [RFC6455][66] |
| 1006   | 异常关闭         | hybi@ietf.org | [RFC6455][67] |
| 1007   | 无效的帧数据     | hybi@ietf.org | [RFC6455][68] |
| 1008   | 违反政策         | hybi@ietf.org | [RFC6455][69] |
| 1009   | 消息太大         | hybi@ietf.org | [RFC6455][70] |
| 1010   | 强制扩展         | hybi@ietf.org | [RFC6455][71] |
| 1011   | 内部服务器错误   | hybi@ietf.org | [RFC6455][72] |
| 1015   | TLS握手          | hybi@ietf.org | [RFC6455][73] |

### 11.8 WebSocket 操作码注册表

该规范根据 [RFC5226][74] 中规定的原则为 WebSocket 协议创建了一个新的 IANA 注册表，用于 WebSocket 操作码。

作为此注册表的一部分，IANA 包含了一下信息：

操作码

操作码表示定义在第 5.2 节中的 WebSocket 帧的帧类型。操作码是一个范围在 0 到 15（含）的数字。

含义

操作码的含义。

关联

请求操作码的规范。

WebSocket 操作码是受到“标准行为”IANA 注册表政策 [RFC5266][75] 限制的。

IANA 已经向注册表中注册了一下初始值。

| 操作码 | 含义         | 关联                                           |
| ------ | ------------ | ---------------------------------------------- |
| 0      | 连续帧       | [RFC6455][76] |
| 1      | 文本帧       | [RFC6455][77] |
| 2      | 二进制帧     | [RFC6455][78] |
| 8      | 连接关闭帧   | [RFC6455][79] |
| 9      | 心跳 Ping 帧 | [RFC6455][80] |
| 10     | 心跳 Pong 帧 | [RFC6455][81] |

### 11.9 WebSocket 帧头 bit 字段注册表

该规范根据 [RFC5226][82] 中规定的原则为 WebSocket 协议创建了一个新的 IANA 注册表，用于 WebSocket 帧头 bit 字段。这个注册表控制分配的 bit 位为第 5.2 节中的 RSV1、RSV2 和 RSV3。

这些 bit 位是保留给将来的版本或者文档中的扩展。

WebSocket 帧头 bit 字段是受到“标准行为”IANA 注册表政策 [RFC5266][83] 限制的。




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
[17]:	https://tools.ietf.org/html/rfc3987#section-5.3.3
[18]:	https://tools.ietf.org/html/rfc3986
[19]:	https://tools.ietf.org/html/rfc3987
[20]:	https://tools.ietf.org/html/rfc6455
[21]:	https://tools.ietf.org/html/rfc5234
[22]:	https://tools.ietf.org/html/rfc3986
[23]:	https://tools.ietf.org/html/rfc3986
[24]:	https://tools.ietf.org/html/rfc3986
[25]:	https://tools.ietf.org/html/rfc3987
[26]:	https://tools.ietf.org/html/rfc3987#section-5.3.3
[27]:	https://tools.ietf.org/html/rfc3986
[28]:	https://tools.ietf.org/html/rfc3987
[29]:	https://tools.ietf.org/html/rfc6455
[30]:	https://tools.ietf.org/html/rfc2817
[31]:	https://tools.ietf.org/html/rfc6455
[32]:	https://tools.ietf.org/html/rfc3864
[33]:	https://tools.ietf.org/html/rfc6455
[34]:	https://tools.ietf.org/html/rfc3864
[35]:	https://tools.ietf.org/html/rfc6455
[36]:	https://tools.ietf.org/html/rfc3864
[37]:	https://tools.ietf.org/html/rfc6455
[38]:	https://tools.ietf.org/html/rfc3864
[39]:	https://tools.ietf.org/html/rfc6455
[40]:	https://tools.ietf.org/html/rfc3864
[41]:	https://tools.ietf.org/html/rfc6455
[42]:	https://tools.ietf.org/html/rfc5226
[43]:	https://tools.ietf.org/html/rfc5226
[44]:	https://tools.ietf.org/html/rfc5226
[45]:	https://tools.ietf.org/html/rfc2616
[46]:	https://tools.ietf.org/html/rfc5226
[47]:	https://tools.ietf.org/html/rfc5226
[48]:	https://tools.ietf.org/html/rfc5226
[49]:	https://tools.ietf.org/html/rfc5226
[50]:	https://tools.ietf.org/html/draft-ietf-hybi-thewebsocketprotocol-00
[51]:	https://tools.ietf.org/html/draft-ietf-hybi-thewebsocketprotocol-01
[52]:	https://tools.ietf.org/html/draft-ietf-hybi-thewebsocketprotocol-02
[53]:	https://tools.ietf.org/html/draft-ietf-hybi-thewebsocketprotocol-03
[54]:	https://tools.ietf.org/html/draft-ietf-hybi-thewebsocketprotocol-04
[55]:	https://tools.ietf.org/html/draft-ietf-hybi-thewebsocketprotocol-05
[56]:	https://tools.ietf.org/html/draft-ietf-hybi-thewebsocketprotocol-06
[57]:	https://tools.ietf.org/html/draft-ietf-hybi-thewebsocketprotocol-07
[58]:	https://tools.ietf.org/html/draft-ietf-hybi-thewebsocketprotocol-08
[59]:	https://tools.ietf.org/html/rfc6455
[60]:	https://tools.ietf.org/html/rfc5226
[61]:	https://tools.ietf.org/html/rfc6455
[62]:	https://tools.ietf.org/html/rfc6455
[63]:	https://tools.ietf.org/html/rfc6455
[64]:	https://tools.ietf.org/html/rfc6455
[65]:	https://tools.ietf.org/html/rfc6455
[66]:	https://tools.ietf.org/html/rfc6455
[67]:	https://tools.ietf.org/html/rfc6455
[68]:	https://tools.ietf.org/html/rfc6455
[69]:	https://tools.ietf.org/html/rfc6455
[70]:	https://tools.ietf.org/html/rfc6455
[71]:	https://tools.ietf.org/html/rfc6455
[72]:	https://tools.ietf.org/html/rfc6455
[73]:	https://tools.ietf.org/html/rfc6455
[74]:	https://tools.ietf.org/html/rfc5226
[75]:	https://tools.ietf.org/html/rfc5226
[76]:	https://tools.ietf.org/html/rfc6455
[77]:	https://tools.ietf.org/html/rfc6455
[78]:	https://tools.ietf.org/html/rfc6455
[79]:	https://tools.ietf.org/html/rfc6455
[80]:	https://tools.ietf.org/html/rfc6455
[81]:	https://tools.ietf.org/html/rfc6455
[82]:	https://tools.ietf.org/html/rfc5226
[83]:	https://tools.ietf.org/html/rfc5226