
## 概述

本文为 WebSocket 协议的第九章，本文翻译的主要内容为 WebSocket 扩展相关内容。

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

## 扩展（协议正文）

WebSocket 客户端可以请求对该规范的扩展，WebSocket 服务端可以接受其中一些或者所有的客户端请求的扩展。服务端禁止响应客户端没有请求过的扩展。如果扩展参数需要在客户端和服务端之间进行协商，这些参数必须根据参数所应用的扩展的规范来选择。

### 9.1 协商扩展

客户端通过 `Sec-WebSocket-Extensions` 请求头字段来请求扩展，请求头字段遵守 HTTP 的规则，它的值是通过 ABNF 定义的。注意这一节使用了 RFC2616 中的 ABNF 语法/规则，包括“implied \*LWS rule”。如果客户端或者服务端在协商扩展时收到了一个不符合下面的 ABNF 规则的值，接收到错误的数据的这一方必须立刻`让 WebSocket 连接失效`。

	Sec-WebSocket-Extensions = extension-list
	extension-list = 1#extension
	extension = extension-token *( ";" extension-param )
	extension-token = registered-token
	registered-token = token
	extension-param = token [ "=" (token | quoted-string) ]
		; 使用带引号的语法变量时，在引号字符后面的变量的值必须符合`token`变量 ABNF规范。

注意，就像其他的 HTTP 请求头字段一样，这个请求头字段可以被切割成几行或者几行合并成一行。因此，下面这两行：

	Sec-WebSocket-Extensions: foo
	Sec-WebSocket-Extensions: bar; baz=2

等价于：

	Sec-WebSocket-Extensions: foo, bar; baz=2

任何一个扩展凭证都必须是一个注册过的凭证（见第 11.4 节）。扩展所使用的任何参数都必须是定义给这个扩展的。注意，客户端只是提议使用它所列出的扩展，除非服务端表示希望使用某个扩展，否则客户端禁止使用它们。

注意扩展的顺序是重要的。多个扩展中的任意的互相作用都可以被定义在这个定义扩展的文档中。在没有此类定义的情况下，客户端在其请求中列出的头字段表示其希望使用的头字段的首选项，其中列出的第一个选项是最可取的。服务器在响应中列出的扩展表示连接实际使用的扩展。如果扩展修改了数据或者帧，对数据的操作顺序应该被假定为和连接开始握手时服务端响应中列举的扩展的顺序相同。

例如，如果有两个扩展“foo”和“bar”，并且服务端发送的头字段`Sec-WebSocket-Extensions`的值为“foo,bar”，那么对数据的操作顺序就是bar(foo(data))，无论是对数据本身的更改（例如压缩），还是对可能“堆叠”的帧的更改。

可接受的扩展标头字段的非规范性示例（请注意，长行被折叠以便于阅读）如下：

	Sec-WebSocket-Extensions: deflate-stream
	Sec-WebSocket-Extensions: mux; max-channels=4; flow-control, deflate-stream
	Sec-WebSocket-Extensions: private-extension

服务端通过包含一个`Sec-WebSocket-Extensions`头字段来接受一个或者多个扩展，这个头字段中包含一个或者多个客户端请求过的扩展。任何扩展参数如何解析，以及对于客户端请求的一组参数，服务端怎样的响应才是有效的，将由每个扩展各自定义。

### 9.2 已知扩展

扩展为实现方式提供了一个机制，用于选择启用额外的协议功能。这个文档中不定义任何扩展，但是实现可以使用单独定义的扩展。

[1]:	https://juejin.im/post/5b12966fe51d450689495e41
[2]:	https://juejin.im/post/5b1a7189e51d45068b496cf0
[3]:	https://juejin.im/post/5b1e6beae51d4506b62cbd64
[4]:	https://juejin.im/post/5b226d716fb9a00e594c5da5
[5]:	https://juejin.im/post/5b2b9850518825748e545d23
[6]:	https://juejin.im/post/5c32f906f265da6136229fac
[7]:	https://juejin.im/post/5c33648b6fb9a049b2220bb7
[8]:	https://juejin.im/post/5c3c3982f265da613f2fafe4
[9]:	https://juejin.im/post/5c4c1d8cf265da61285a759d