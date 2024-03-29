
## 概述

本文为 WebSocket 协议的第十章，本文翻译的主要内容为 WebSocket 安全性相关内容。

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

## 10 安全性考虑（协议正文）

这一章描述了一些 WebSocket 协议的可用的安全性考虑。这一章的小节描述了这些特定的安全性考虑。

### 10.1 非浏览器客户端

WebSocket 协议防止在受信任的应用例如 Web 浏览器中执行的恶意 JavaScript 代码，例如通过检查`Origin`头字段（见下面）。见第 1.6 节去了解更多详情。这种假设在更有能力的客户端的情况下不成立。

这个协议可以被网页中的脚本使用，也可以通过宿主直接使用。这些宿主是代表自己的利益的，因此可以发送假的`Origin`头字段来欺骗服务端。因此服务端对于他们正在和已知的源的脚本直接通信的假设需要消息，并且必须认为他们可能通过没有预期的方式访问。特别地，服务端不应该相信任何输入都是有效的。

示例：如果服务端使用输入的内容作为一部分的 SQL 查询语句，所有的输入文本都必须在传递给 SQL 服务器时进行编码，以免服务端受到 SQL 注入攻击。

### 10.2 源考虑

只处理特定站点，不打算处理任何 Web 页面的数据服务器应该验证`Origin`字段是否是他们预期的。如果服务端收到的源字段是不接受的，那么他应该通过包含 HTTP 禁止状态码为 403 的请求响应作为 WebSocket 握手的响应。

当不信任的一方是 JavaScript 应用作者并存在受信任的客户端中运行时，`Origin`字段可以避免出现这种攻击的情况。客户端可以连接到服务端，通过协议中的`Origin`字段，确定是否开放连接的权限给 JavaScript 应用。这么做的目的不是组织非浏览器应用建立连接，而是保证在受信任的浏览器中可能运行的恶意 JavaScript 代码并不会构建一个假的 WebSocket 握手。

### 10.3 基础设施攻击（添加掩码）

除了终端可能会成为通过 WebSocket 被攻击的目标之外，网络基础设施的另外一部分，例如代理，也有可能是攻击的对象。

这个协议发展后，通过一个实验验证了部署在外部的缓存服务器由于一系列在代理上面的攻击导致投毒。一般形式的攻击就是在攻击者控制下建立一个与服务端的连接，实现一个与 WebSocket 协议建立连接相似的 HTTP UPGRADE 连接，然后通过升级以后的连接发送数据，看起来就像是针对已知的特定资源（在攻击中，这可能类似于广泛部署的脚本，用于跟踪广告服务网络上的点击或资源）进行 GET 请求。远端服务器可能会通过一些看上去像响应数据的来响应假的 GET 请求，然后这个响应就会按照非零百分比的已部署中介缓存，因此导致缓存投毒。这个攻击带来的影响就是，如果一个用户可以正常的访问一个攻击者控制的网网站，那么攻击者可以针对这个用户进行缓存投毒，在相同缓存的后面其他用户会运行其他源的恶意脚本，破坏 Web 安全模型。

为了避免对中介服务的此类攻击，使用不符合 HTTP 的数据帧中为应用程序的数据添加前缀是不够的，我们不可能详细的检查和测试每一个不合标准的中介服务有没有跳过这种非 HTTP 帧，或者对帧载荷处理不正确的情况。因此，采用的防御措施是对客户端发送给服务端的所有数据添加掩码，这样的话远端的脚本（攻击者）就不能够控制发送的数据如何出现在线路上，因此就不能够构造一条被中介误解的 HTPT请求。

客户端必须为每一帧选择一个新的掩码值，使用一个不能够被应用预测到的算法来进行传递数据。例如，每一个掩码值可以通过一个加密强随机数生成器来生成。如果相同的值已经被使用过或者已经存在一种方式能够判断出下一个值如何选择时，攻击这个可以发送一个添加了掩码的消息，来模拟一个 HTTP 请求（通过在线路上接收攻击者希望看到的消息，使用下一个被使用的掩码值来对数据进行添加掩码，当客户端使用它时，这个掩码值可以有效地反掩码数据）。

当从客户端开始传递第一帧时，这个帧的有效载荷（应用程序提供的数据）就不能够被客户端应用程序修改，这个策略是很重要的。否则，攻击者可以发送一个都是已知值（例如全部为 0）的初始值的很长的帧，计算收到第一部分数据时使用过的掩码，然后修改帧中尚未发送的数据，以便在添加掩码时显示为 HTTP 请求。（这与我们在之前的段落中描述的使用已知的值和可预测的值作为掩码值，实际上是相同的问题。）如果另外的数据已经发送了，或者要发送的数据有所改变，那么新的数据或者修改的数据必须使用一个新的数据帧进行发送，因此也需要选择一个新的掩码值。简短来说，一旦一个帧的传输开始后，内容不能够被远端的脚本（应用）修改。

受保护的威胁模型是客户端发送看似HTTP请求的数据的模型。因此，从客户端发送给服务端的频道数据需要添加掩码值。从服务端到客户端的数据看上去像是一个请求的响应，但是，为了完成一次请求，客户端也需要可以伪造请求。因此，我们不认为需要在双向传输上添加掩码。（服务端发送给客户端的数据不需要添加掩码）

尽管通过添加掩码提供了保护，但是不兼容的 HTTP 代理仍然由于客户端和服务端之间不支持添加掩码而受到这种类型的攻击。

### 10.4 指定实现的限制

在从多个帧重新组装后，对于帧大小或总消息大小具有实现和必须避免自己超过相关的多平台特定限制带来的影响。（例如：一个恶意的终端可能会尝试耗尽对端的内存或者通过发送一个大的帧（例如：大小为 2 \*\* 60）或发送一个长的由许多分片帧构成的流来进行拒绝服务攻击）。这些实现应该对帧的大小和组装过后的包的总大小有一定的限制。

### 10.5 WebSocket 客户端认证

这个协议在 WebSocket 握手时，没有规定服务端可以使用哪种方式进行认证。WebSocket 服务器可以使用任意 HTTP 服务器通用的认证机制，例如： Cookie、HTTP 认证或者 TLS 认证。

### 10.6 连接保密性和完整性

连接保密性是基于运行 TLS 的 WebSocket 协议（wss 的 URLs）。WebSocket 协议实现必须支持 TLS，并且应该在与对端进行数据传输时使用它。

如果在连接中使用 TLS，TLS带来的连接的收益非常依赖于 TLS 握手时的算法的强度。例如，一些 TLS 的加密算法不提供连接保密性。为了实现合理登记的保护措施，客户端应该只使用强 TLS 算法。“Web 安全：用户接口指南”（[W3C.REC-wsc-ui-20100812][11]）讨论了什么是强 TLS 算法。[RFC5246][12] 的[附录 A.5][13]和[附录 D.3][14]提供了另外的指导。

### 10.7 处理无用数据

传入的数据必须经过客户端和服务端的认证。如果，在某个时候，一个终端面对它无法理解的数据或者违反了这个终端定义的输入安全规范和标准，或者这个终端在开始握手时没有收到对应的预期值时（在客户端请求中不正确的路径或者源），终端应该关闭 TCP 连接。如果在成功的握手后收到了无效的数据，终端应该在进入`关闭 WebSocket`流程前，发送一个带有合适的状态码（第 7.4 节）的关闭帧。使用一个合适的状态码的关闭帧有助于诊断这个问题。如果这个无效的数据是在 WebSocket 握手时收到的，服务端应该响应一个合适的 HTTP 状态码（[RFC2616][15]）。

使用错误的编码来发送数据是一类通用的安全问题。这个协议指定文本类型数据（而不是二进制或者其他类型）的消息使用 UTF-8 编码。虽然仍然可以得到长度值，但实现此协议的应用程序应使用这个长度来确定帧实际结束的位置，发送不合理的编码数据仍然会导致基于此协议构建的应用程序可能会导致从数据的错误解释到数据丢失或潜在的安全漏洞出现。

### 10.8 在 WebSocket 握手中使用 SHA-1

在这个文档中描述的 WebSocket 握手协议是不依赖任意 SHA-1 的安全属性，流入抗冲击性和对第二次前映像攻击的抵抗力（就像 [RFC4270][16] 描述的一样）。



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
[11]:	https://tools.ietf.org/html/rfc6455#ref-W3C.REC-wsc-ui-20100812
[12]:	https://tools.ietf.org/html/rfc5246
[13]:	https://tools.ietf.org/html/rfc6455#appendix-A.5
[14]:	https://tools.ietf.org/html/rfc6455#appendix-D.3
[15]:	https://tools.ietf.org/html/rfc2616
[16]:	https://tools.ietf.org/html/rfc4270