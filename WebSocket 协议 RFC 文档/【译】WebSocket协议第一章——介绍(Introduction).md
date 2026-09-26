# 概述

本文为WebSocket协议的第一章，本文翻译的主要内容为针对整个WebSocket进行一个简单而又全面的介绍。通过这篇文章我们能够对WebSocket有一个整体的大致了解。

# 1 介绍

本章为协议正文内容的第一章（Introduction）。

## 1.1 背景

此章节为非规范章节。

在历史上，创建一个需要在客户端和服务端之间进行双向数据通信的Web应用（例如IM应用和游戏应用）需要滥用HTTP来轮询服务端获取更新，同时通过另外的[HTTP请求][1]向上游发送通知。

这个方法有许多的问题：

- 服务端被迫为每个客户端使用多个不同的底层TCP连接进行交互：一个用来向客户端发送数据，而每收到一条消息都要使用一个新的连接。
- 有线传输协议（wire protocol）开销较大，每一个客户端到服务端的消息都有一个HTTP头。
- 客户端脚本必须维护一个从发送连接到接收连接的映射表来追踪响应。

一个简单的解决方案是使用单个TCP连接来进行双向数据传输。这就是WebSocket提供的能力。结合WebSocket的[API][2]，它能够提供一个可以替代HTTP轮询的方法来满足Web页面和远端服务器的双向数据通信。

相同的技术可以被用到许多的Web应用：游戏、股票应用、多人协作应用、与后端服务实时交互的用户接口等。

WebSocket协议设计的目的是取代已经存在的、为了利用现有基础服务（如代理、过滤器、认证服务）而使用HTTP作为传输层的双向通信技术。这些技术是在效率和可靠性之间权衡后实现的，因为HTTP协议最初并[不是用来做双向数据通信的][3]。WebSocket协议尝试实现基于现有的HTTP基础服务来实现在现有环境中双向通信技术的目标；所以，即使这意味着在现有环境中会有一些复杂性，它在设计中仍然使用了HTTP的80和443端口，以及支持HTTP代理。然而，这个设计并没有限制WebSocket只能使用HTTP端口，在以后的实现中也可以在专用的端口上使用一个更简单的握手方式，而不需要改动整个协议。最后一点很重要，因为双向消息的通信方式不是很符合标准HTTP的模式，可能导致在某些组件中出现异常的负载。

## 1.2 协议概览

此节为非规范章节。

这个协议有两部分：握手和数据传输。

来自客户端的握手数据如下所示：

```http
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Origin: http://example.com
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
```

服务端的握手响应如下所示：

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat
```

客户端请求的第一行（leading line）遵循了HTTP请求行的格式。

服务端的第一行（leading line）遵循了HTTP状态行的格式。

HTTP请求行和状态行的规范定义在[RFC2616][4]。

在这两种情况中，第一行下面是一组无序的header字段。这些header字段的含义定义在本文的[第四节][5]。另外的header字段如[cookies][6]，也有可能存在。格式和解析头信息被定义在了[RFC2616][7]。

当客户端和服务端都发送了他们的握手协议，并且当握手已经成功，那么数据传输就开始了。这是一个双方都可以独立发送任意数据的双向通信渠道。

在握手成功以后，客户端和服务端来回传输数据的概念单位，我们在规范中称为消息（messages）。在传输中，一条消息由一个或者多个帧组成。WebSocket中的消息不需要对应特定网络层中的帧，一条分片的消息可能会被中间人合并或者拆分。

帧有关联的类型。同一条消息的每一帧都包含相同类型的数据。通常来说，它可以是文本数据（UTF-8编码）、二进制数据（留给应用解析的数据）和控制帧数据（不是用来传输数据，而是用来作为协议层的特定符号，如关闭连接帧）。当前版本的协议定义了6种帧类型并且预留了10个保留类型。


## 1.3 开始握手

此节为非规范章节。

开始握手的目的是与基于HTTP的服务端软件和中介兼容，从而使同一个端口既能够满足HTTP客户端来与服务进行交互，又能够满足WebSocket客户端与服务进行交互。为此，WebSocket客户端的握手是一个基于HTTP的升级请求：

```http
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Origin: http://example.com
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
```

遵照[RFC2616][8]，客户端在握手过程中发送的header字段可能是乱序的，所以收到的header字段的顺序不同也没有太大影响。

GET方法的请求URI（Request-URI）是用于定义WebSocket连接的终端，允许同一个IP对多个域名提供服务，也允许同一个服务器提供多个WebSocket终端。

客户端在每一个握手的`Host`header里面包含了一个主机域名。所以客户端和服务端都可以校验双方使用的是否是同一个主机。

另外的header字段是用来确定WebSocket协议的选项。这个版本中提供的特定选项是子协议选择(`Sec-WebSocket-Protocol`)、客户端支持的扩展列表（`Sec-WebSocket-Extensions`）、`Origin`header字段等。请求header字段`Sec-WebSocket-Protocol`可以用来标识哪些子协议（基于WebSocket的应用高层协议）是客户端可以支持的。服务端会从中选择零个或者一个支持的协议并且在响应握手中输出它选择的那个协议。

```http
Sec-WebSocket-Protocol: chat
```

服务端也可以设置cookie相关字段来设置cookie相关属性，具体文档见[RFC6265][9]。

## 1.4 结束握手

此节为非规范章节。

结束握手远比连接握手简单。

任何一端都可以发送一个包含特定关闭握手的控制帧数据（详情见5.5.1节）。收到此帧后，如果另一端还没有发送过结束帧，就会发送一个结束帧作为响应。收到另一端的结束帧后，最开始发送控制帧的端就可以安全地关闭此连接，因为它知道不会再有后续的数据到来。

在发送了一个表明连接需要被关闭的控制帧后，这一端不会再发送任何的数据；在收到一个表明连接需要被关闭的控制帧后，这一端会丢弃此后收到的所有数据。

两端同时发起结束握手也是安全的。

这个结束握手的目标是补充TCP的结束握手（FIN/ACK），而这是因为TCP结束握手在端与端之间并不一定可靠，尤其是有代理和其他的网络中介时会变得不可靠。

通过发送关闭帧并等待接收另一端的响应关闭帧，在某些情况下可以避免数据的不必要丢失。例如，在某些平台中，如果一个socket在接收队列有数据时被关闭，会发送一个RST包，尽管数据还在等待被读取，这也会导致接收到RST的一方数据接收失败。

## 1.5 设计哲学

此节为非规范章节。

WebSocket协议的设计原则是将帧结构（framing）最小化，唯一存在的帧结构就是为了使这个协议基于帧而不是基于流，并且能够区分Unicode文本帧和二进制帧。元数据应该由应用层在WebSocket之上进行分层，就像应用层（例如HTTP）在TCP之上对元数据进行分层一样。

从概念上来看，WebSocket层是基于TCP实现的，增加了以下的内容：

- 增加了一个基于浏览器的同源策略模型
- 增加了一个地址和协议命名机制用以在同一个端口上支持多个服务，在同一个IP地址支持多个主机名
- 在TCP协议之上构建一层帧机制，以回归TCP所基于的IP包机制，但是没有长度限制
- 包含一个设计用于处理有代理和其他网络中介的情况的额外的结束握手协议

除此之外，WebSocket没有增加任何东西。基本上WebSocket的目标是在Web的约束条件下向脚本提供尽可能接近原生TCP的能力。它的设计同时使得WebSocket服务器可以和HTTP服务器共用一个端口，因为它的握手是一个有效的HTTP升级请求。大家也可以使用其他协议来建立从客户端到服务端的消息通信，但WebSocket的协议的目的是为了提供一个相对简单的、可以和HTTP以及已部署的HTTP基础设施（如代理）共存的协议。在考虑安全因素的前提下，这个协议在能够安全地与这些基础设施一起使用的范围内尽可能地接近TCP，并且做了一些有针对性的补充来简化使用、让简单的事情保持简单（例如增加了消息语义）。

这个协议具有可扩展性，未来的版本可能会引入一些新的概念如多路复用。

## 1.6 安全模型

此节为非规范章节。

当WebSocket协议在web网页中应用时，WebSocket协议在Web页面与WebSocket服务器建立连接时使用基于web浏览器的同源策略模型。当然，当WebSocket协议在一个特定的客户端（不是web浏览器里面的网页）直接使用时，同源策略模型就没有意义了，因为客户端可以提供任意的源字符串。

该协议被设计为无法与已经存在的如SMTP（[RFC5321][10]）和HTTP协议的服务器建立连接，如果需要的话，HTTP服务器可以选择支持该协议。这是通过严格而精细的握手过程，以及限制在握手完成之前可以插入连接的数据来实现的（因此限制了服务器可能受到的影响）。

同样，当其他协议（尤其是HTTP）的数据被发送到WebSocket服务器时，也无法建立连接。例如，一个HTML“表单”被提交给一个WebSocket服务器时就可能发生这种情况。这主要是通过要求服务端证明它读取了握手数据来实现的，而只有当握手数据包含只能由WebSocket客户端发送的特定部分时，服务端才能做到这一点。尤其是在编写这个规范时，攻击者无法在Web浏览器中仅使用HTML和JavaScript APIs（如XMLHttpRequest）来设置以`Sec-`开头的字段。

## 1.7 与TCP和HTTP的关系

此节为非规范章节。

WebSocket协议是独立的基于TCP的协议。它和HTTP的唯一关系是它的握手会被HTTP服务器解释为一个升级请求。

WebSocket默认使用80端口进行连接，而基于TLS（[RFC2818][11]）的WebSocket连接是基于443端口的。

## 1.8 建立连接

此节为非规范章节。

当建立了一个和HTTP服务器共享端口的连接时（这种情况很有可能发生在与80和443端口通信上），这个连接将会给HTTP服务器发送一个常规的GET请求来进行升级。在一个IP地址和一个单一的服务器来应对单一主机名的通信这种相对简单的设置上，基于WebSocket协议的系统可以通过一个更加实用的方法来进行部署。在更复杂的设置（例如负载均衡和多服务器），与HTTP服务器分开的专属的WebSocket连接集群可能更加易于管理。在编写这个规范时，我们应该知道在80端口和443端口建立WebSocket连接的成功率是不同的，在443端口上面建立的连接很明显更容易成功，尽管这可能随着时间的变化而改变。

## 1.9 使用WebSocket协议的子协议

客户端可以通过在握手阶段中的`Sec-WebSocket-protocol`字段来请求服务端使用指定的子协议。如果指定了这个字段，服务器需要包含相同的字段，并且从客户端提供的子协议之中选择一个值作为建立连接的响应。

子协议的名称可以按照[第11.5节][12]的方法进行注册。为了避免潜在的冲突，推荐使用包含子协议创建者域名（ASCII形式）的名称作为子协议名。例如，Example Corporation创造了在Web上通过多个服务器实现的一个聊天子协议（Chat subprotocol），他们可以叫做`chat.example.com`。如果Example Organization创造了他们与之竞争的子协议叫做`chat.example.org`，这两个子协议可以被服务器同时实现，服务器可以根据客户端来动态地选择使用哪一个子协议。

子协议也可以通过修改名字的方式来进行不向后兼容的版本升级，例如：将`bookings.example.net`改为`v2.bookings.example.net`。WebSocket客户端会把这些子协议视为完全不同的子协议。向后兼容的版本控制可以通过复用相同的子协议字符串，并小心设计实际的子协议来支持这种扩展性来实现。

# 总结

本文通过对WebSocket进行了一个全面的大致介绍，能够让大家对于WebSocket相关协议内容有一个初步的理解。

后续会持续对WebSocket协议进行翻译，有兴趣了解的同学可以持续关注下。




[1]:	https://tools.ietf.org/html/rfc6202
[2]:	https://tools.ietf.org/html/rfc6455#ref-WSAPI
[3]:	https://tools.ietf.org/html/rfc6202
[4]:	https://tools.ietf.org/html/rfc2616
[5]:	https://tools.ietf.org/html/rfc6455#section-4
[6]:	https://tools.ietf.org/html/rfc6265
[7]:	https://tools.ietf.org/html/rfc2616
[8]:	https://tools.ietf.org/html/rfc2616
[9]:	https://tools.ietf.org/html/rfc6265
[10]:	https://tools.ietf.org/html/rfc5321
[11]:	https://tools.ietf.org/html/rfc2818
[12]:	https://tools.ietf.org/html/rfc6455#section-11.5
