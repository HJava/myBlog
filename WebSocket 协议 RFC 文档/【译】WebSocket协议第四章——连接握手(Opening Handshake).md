# 概述

本文为WebSocket协议的第四章，本文翻译的主要内容为WebSocket建立连接开始握手的内容，主要包含了客户端和服务端握手的内容，以及双方如何处理相关字段和逻辑。

# 4 开始握手（协议正文）

## 4.1 客户端要求

为了建立一个WebSocket连接，客户端需要建立一个连接并且发送一个在本节中定义的握手协议。连接最初状态为`CONNECTING`。客户端需要提供一个第三章讨论过的主机（host）、端口（port）、资源名称（resource name）和安全标记（secure）字段以及可被使用的一个协议（protocol）和扩展（extensions）列表。另外，如果客户端是一个Web浏览器，还需要提供源（origin）字段。

客户端在一个受控制的环境内运行，如使用特定运营商的手机浏览器，可能会断开连接切换到其他的运营商。在这种情况下，我们需要考虑包括手机软件和相关运营商在内的指定客户端。

当客户端通过一系列的配置字段（主机（host）、端口（port）、资源名称（resource name）和安全标记（secure））以及一个可被使用的协议（protocol）和扩展（extensions）列表来建立一个WebSocket连接，它一定会通过发送一个握手协议，并且受到一个服务端的握手响应来建立一条连接。建立连接具体需要哪些东西，在开始握手的时候会发送哪些字段，如何处理解读服务端的的响应都会在这一部分得到解答。在下面的内容中，我们会使用到第三章定义的一些术语如主机（host）和安全（secure）字段。

1. WebSocket的URI部分传递的字段（主机（host）、端口（port）、资源名称（resource name）和安全标记（secure））必须是在第三章WebSocket URIs部分指定过的有效字段，如果任意部分是无效字段，那么客户端一定会在接下来的步骤中关闭连接。
2. 如果客户端有一条通过远端主机（IP地址）定义的主机和端口定义的已经建立连接的WebSocket连接，即使这个远端主机被定义为了其他的名字，这个客户端也必须等到当前的这条连接建立成功或者失败才能建立连接。客户端最多有一条连接可以处于`CONNECTING`状态。如果多个连接尝试同时与一个相同的IP地址建立连接，客户端必须把他们进行排序，所以只能有一个连接执行下面的步骤。

	如果客户端不能够确定远程主机的IP地址（例如所有的请求都通过一个自己执行DNS查询的代理），那么客户端必须基于此假设每一个主机名都对应着不同的远端主机，因此客户端应该限制同时连接的总数目在一个比较合理的小数目上（例如：客户端可能允许同时跟`a.example.com`和`b.example.com`这两个地址建立连接，但是如果同时和主机建立三十个连接，这可能是不允许的）。例如：在Web浏览器环境下，客户端需要考虑在用户打开的多个tab页中设置一个同时建立连接的数目限制。

	注意：这个限制使得脚本仅仅通过创建大量的WebSocket连接来进行拒绝服务攻击变得更难了。服务端可以在关闭连接前就停止攻击，从而进一步减小负载，这样会减少客户端的重连率。

	注意：客户端可以与单个主机建立的WebSocket连接数量是没有限制的。当建立的连接过多时，服务端可以拒绝和主机/IP地址建立的连接，同时服务端在负载过高时也可以主动断开占用资源的连接。

3. 使用代理：如果客户端在使用WebSocket协议来连接特定的主机和端口时使用了配置的代理，那么客户端应该连接到那个代理并且通过这个代理去和指定的主机和端口建立一个TCP连接。

	例如：如果客户端使用了全局的HTTP代理，那么如果尝试和example.com的80端口建立连接，那么久可能会发送下面的字段给代理服务器：

	```http
	CONNECT example.com:80 HTTP/1.1
	Host: example.com
	```

		如果有密码字段的话，那么可能如下所示： 

		CONNECT example.com:80 HTTP/1.1
		Host: example.com
		Proxy-authorization: Basic ZWRuYW1vZGU6bm9jYXBlcyE=

	如果客户端没有配置代理，那么就应该会和给定的主机和端口直接建立一条TCP连接。

	注意：如果可以，实现不暴露明显界面的来给WebSocket选择与其他代理分开的代理推荐使用SOCKS5（[RFC1928][1]）代理供WebSocket连接，如果不行的话，使用配置了HTTPS连接的代理优于使用HTTP连接的代理。

	为了自动配置脚本，传递参数的URI必须包含定义在第三节WebSocket URI中的主机（host）、端口（port）、资源名称（resource name）和安全（secure）字段。

	注意：WebSocket协议可以根据定义的规范配置到代理自动配置脚本（"ws"代表非加密连接，"wss"代表加密连接）。

4. 如果连接没有被打开，或者由于直连失败或者代理返回了一个错误，那么客户端必须断开WebSocket连接，并且停止重试连接。
5. 如果安全（secure）字段存在，客户端必须在连接建立以后、发送握手数据之前进行TLS握手。如果TLS握手失败（比如服务端正数没有验证通过），那么客户端必须断开WebSocket连接。否则，所有后续的在此频道上面的数据通信都必须在加密的通道中传输。


	客户端在TLS握手时必须使用服务器名称指示扩展（SNI，Server Name Indication）。

一旦到服务端的连接被建立了（包括通过一个代理或者通过一个TLS加密通道），客户端必须发送一个开始握手的数据包给服务端。这个数据包由一个HTTP升级请求构成，包含一系列必须的和可选的header字段。握手的具体要求如下所示：

1. 握手必须是一个在[RFC2616][2]指定的有效的HTTP请求。
2. 这个请求方法必须是GET，而且HTTP的版本至少需要1.1。

	例如：如果WebSocket的URI是"ws://example.com/chat"，那么发送的请求头第一行就应该是"GET /chat HTTP/1.1"。

3. 请求的"Request-URI"部分必须与第三章中定义的资源名称（resource name）匹配，或者必须是一个http/https绝对路径的URI，当解析URI时，有一个资源名称（resource name）、主机（host）和端口（port）与相对应的ws/wss匹配。
4. 请求必须包含一个`Host`header字段，它包含了一个主机（host）字段加上一个紧跟在":"之后的端口（port）字段（如果端口不存在则使用默认端口）。
5. 这个请求必须包含一个`Upgrade`header字段，它的值必须包含"websocket"。
6. 请求必须包含一个`Connection`header字段，它的值必须包含"Upgrade"。
7. 请求必须包含一个名为`Sec-WebSocket-Key`的header字段。这个header字段的值必须是由一个随机生成的16字节的随机数通过base64（见[RFC4648的第四章][3]）编码得到的。每一个连接都必须随机的选择随机数。

	注意：例如，如果随机选择的值的字节顺序为0x01 0x02 0x03 0x04 0x05 0x06 0x07 0x08 0x09 0x0a 0x0b 0x0c 0x0d 0x0e 0x0f 0x10，那么header字段的值就应该是"AQIDBAUGBwgJCgsMDQ4PEC=="。

8. 如果这个请求来自一个浏览器，那么请求必须包含一个`Origin`header字段。如果请求是来自一个非浏览器客户端，那么当该客户端这个字段的语义能够与示例中的匹配时，这个请求也可能包含这个字段。这个header字段的值为建立连接的源代码的源地址ASCII序列化后的结果。通过[RFC6454][4]可以知道如何构造这个值。

	例如，如果在www.example.com域下面的代码尝试与ww2.example.com这个地址建立连接，那么这个header字段的值就应该是"http://www.example.com"。

9. 这个请求必须包含一个名为`Sec-WebSocket-Version`的字段。这个header字段的值必须为13。

	注意：尽管这个文档草案的版本（09，10，11和12）都已经发布（这些协议大部分是编辑上的修改和澄清，而不是对无线协议的修改），9，10，11，12这四个值不被认为是有效的`Sec-WebSocket-Version`的值。这些值被IANA保留，但是没有被用到过，以后也不会被使用。

10. 这个请求可能会包含一个名为`Sec-WebSocket-Protocol`的header字段。如果存在这个字段，那么这个值包含了一个或者多个客户端希望使用的用逗号分隔的根据权重排序的子协议。这些子协议的值必须是一个非空字符串，字符的范围是U+0021到U+007E，但是不包含其中的定义在[RFC2616][5]中的分隔符，并且每个协议必须是一个唯一的字符串。ABNF的这个header字段的值是在[RFC2616][6]定义了构造方法和规则的1#token。
11. 这个请求可能包含一个名为`Sec-WebSocket-Extensions`字段。如果存在这个字段，这个值表示客户端期望使用的协议级别的扩展。这个header字段的具体内容和格式具体见9.1节。
12. 这个请求可能还会包含其他的文档中定义的header字段，如cookie（[RFC6265][7]）或者认证相关的header字段如`Authorization`字段（[RFC2616][8]）。

一旦客户端的握手请求发送出去，那么客户端必须在发送后续数据前等待服务端的响应。客户端必须通过以下的规则验证服务端的请求：

1. 如果客户端收到的服务端返回状态码不是101，客户端需要处理每个HTTP请求的响应。特别的是，客户端需要在收到401状态码的时候可能需要进行验证；服务端可能会通过3xx的状态码来将客户端进行重定向（但是客户端不要求遵守这些）等。否则，遵循下面的步骤。
2. 如果客户端收到的响应缺少一个`Upgrade`header字段或者`Upgrade`header字段包含一个不是"websocket"的值（该值不区分大小写），那么客户端必须关闭连接。
3. 如果客户端收到的响应缺少一个`Connection`header字段或者`Connection`header字段不包含"Upgrade"的值（该值不区分大小写），那么客户端必须关闭连接。
4. 如果客户端收到的`Sec-WebSocket-Accept`header字段或者`Sec-WebSocket-Accept`header字段不等于通过`Sec-WebSocket-Key`字段的值（作为一个字符串，而不是base64解码后）和"258EAFA5-E914-47DA-95CA-C5AB0DC85B11"串联起来，忽略所有前后空格进行base64 SHA-1编码的值，那么客户端必须关闭连接。
5. 如果客户端收到的响应包含一个`Sec-WebSocket-Extensions`header字段，并且这个字段使用的extension值在客户端的握手请求里面不存在（即服务端使用了一个客户端请求中不存在的值），那么客户端必须关闭连接。（解析这个header字段来确定使用哪个扩展在9.1节中有讨论。）
6. 如果客户端收到的响应包含一个`Sec-WebSocket-Protocol`header字段，并且这个字段包含了一个没有在客户端握手中出现的子协议（即服务端使用了一个客户端请求中子协议字段不存在的值），那么客户端必须关闭连接。

如果客服务端的响应没有符合定义在这一节和4.2.2节中的服务端握手响应定义的要求，那么客户端也会断开连接。

请注意，根据[RFC2616][9]，所有的header字段名称在HTTP请求和HTTP请求响应中都是不区分大小写的。

如果服务端的响应通过了上述的验证过程，那么WebSocket就已经建立连接了，并且WebSocket的连接状态也到了`OPEN`状态。使用的扩展被定义为一个字符串（可能为空），它是在服务端响应握手时候提供的`Sec-WebSocket-Extensions`字段的值，如果这个header字段在握手响应中不存在，那么就是一个空值。使用的子协议值是在服务端响应握手中提供的`Sec-WebSocket-protocol`字段的值，如果服务端响应握手时没有这个header字段，那么这个值也为空。另外，如果服务端握手响应时设置了任何cookie的header字段（定义在[RFC6265][10]）,这些cookie被称为在服务端响应握手时设置的cookie（Cookies Set During the Server's Opening Handshake）。

## 4.2 服务端要求

服务端可以将连接的管理挂载到其他的网络代理上，如负载均衡器或者反向代理。在这种情况下，这篇规范对于服务端的目标是包含从第一个设备从建立到断开连接的TCP连接周期到服务端接受请求，发送响应的所有服务器的基础设施部分。

示例：一个数据中心可能有一个响应WebSocket握手请求的服务器，但是它将收到的数据帧都通过连接传递给另一个服务器来处理。在本文档中，"服务端（server）"包含这两者。

### 4.2.1 解析客户端的握手协议

当客户端开始一个WebSocket连接时，他会发送一个开始握手协议。为了获得必要的信息来保证服务端的握手响应，服务端必须解析这个客户端这部分的握手协议。

客户端的握手协议包含以下几部分。当服务的收到一个握手请求，发现客户端并没有发送一个符合以下内容的握手协议（注意在[RFC2616][11]中的每一项，header字段的顺序是不重要的），包括但不限于在握手协议中有不合法的ANBF语法，服务端必须立即停止处理客户端的握手请求并且在响应中返回一个表示错误的HTTP错误码（如400 Bad Request）。

1. 一个HTTP/1.1或者跟高版本的GET请求，包含一个在第三章定义的应该被解析为资源名称（resource name）"Request-URI"字段（或者包含资源名称（resource name）的HTTP/HTTPS绝对路径）。
2. 包含服务端权限的`Host`header字段。
3. 不区分大小写的值为"websocket"的`Upgrade`header字段。
4. 不区分大小写的值为"Upgrade"的`Connection`header字段。
5. 值为base64编码（见[RFC4648的第四章][12]）后长度为16字节的`Sec-WebSocket-Key`header字段。
6. 值为13的`Sec-WebSocket-Version`header值。
7. 可选的`Origin`header字段。所有的浏览器都会发送这个字段。缺少此字段的连接不应该认为是来自浏览器。
8. 可选的`Sec-WebSocket-Protocol`header字段，对应的值为客户端支持的子协议，根据权重进行排序。
9. 可选的`Sec-WebSocket-Extensions`header字段，对应的值为客户端可以使用的扩展。这个字段具体内容会在第9.1节再进行讨论。
10. 可选的其他字段，如使用cookie或者服务器请求认证的字段。不识别的header字段会依据[RFC2616][13]中内容被忽略。

### 4.2.2 发送服务端握手响应请求

当客户端和服务端建立了一个WebSocket连接，服务端也必须完成接受连接的下面说明的步骤，并且发送一个服务端握手响应。

1. 如果是一条建立在HTTPS（HTTPS+TLS）端口的连接，通过这个链接完成TLS握手过程。如果这次握手失败（例如，客户端在"server\_name"扩展中制定了主机名，但是服务端没有这个主机），那么关闭这条连接；否则，后续这个连接的所有的数据传递（包括服务端握手响应）都必须使用一个加密的通道。
2. 服务端可以选择额外的客户端认证，例如，通过返回401状态码和在[RFC2616][14]说明的相对应的`WWW-Authenticate`header字段。
3. 服务端可能通过使用3xx的状态码（见[RFC2616][15]）来重定向客户端。注意这个步骤可以发生在上面说到的认证之前、之后或者和认证一起。
4. 构造以下信息：
	 
	源（`origin`）

	- `Origin`header字段在客户端的握手请求中表示建立连接的脚本属于哪一个源。这个源信息被序列化为ASCII，并且转换为小写。服务端可以使用这个信息来作为判断是否接受这个链接的部分参考内容。如果服务端没有过滤源，那么他会接受任意源的连接。如果服务端没有接受这个连接，那么它必须返回一个对应的HTTP错误码（如403 Forbidden）并且终端这一节描述的WebSocket握手过程。更多详情可以阅读第十章。
	关键值（`key`）
	- `Sec-WebSocket-Key`header字段在客户端的握手请求中表示一个长度为16字节的base64编码的值。这个编码后的值是用于服务端握手的创建过程，用来表示接受了这个连接。服务端没有必要对`Sec-WebSocket-Key`值进行解码。
	- 版本（`version`）
	`Sec-WebSocket-Version`header字段在客户端握手请求中表示了客户端建立连接使用的WebSocket协议版本。如果这个版本和服务端的版本没有匹配上，那么服务端必须中断本章说的WebSocket连接，并且发送一个对应的HTTP错误码（例如426 Upgrade Required），同时返回一个`Sec-WebSocket-Version`header字段用来标识服务端能够识别的版本号。
	- 资源名称（`resource name`）
	服务端提供的服务标识符。如果这个服务端提供多种服务，那么这个值应该是来自客户端握手请求中的GET方法中的"Request-URI"字段。如果请求的服务不支持，那么服务端必须发送一个相对应的HTTP错误码（例如404 Not Found）并且中断WebSocket连接。
	- 子协议（`subprotocol`）
	服务端准备使用的代表子协议的单个值或者为空。这个值必须选择客户端握手协议中由`Sec-WebSocket-Protocol`字段中提供的值，服务端会在这个连接中使用此值（任意）。如果客户端握手协议中没有包含这个字段或者服务端不支持客户端请求中提供的任意一个子协议，那么这个值只能为空。没有此header值就表明该值为空（这意味着服务端可以不选择客户端传递的任意一个子协议，禁止在响应请求中添加一个`Sec-WebSocket-Protocol`字段）。空字符串与空值不同，并且空值对于此字段来说是一个不合法值。ABNF对于整个字段的定义和构造规则可以见[RFC2616][16]。
	- 扩展（`extensions`）
	表示一个服务端准备使用的协议级扩展列表（可能为空）。如果服务端支持多种扩展，那么这个值必须是客户端握手中已有的数值，是从`Sec-WebSocket-Extensions`字段中取一到多个值。该字段不存在时则表示此值为空。空字符串与空值不同。客户端没有列举的扩展禁止被
	使用。应该选择哪些值和如何进行解析可以见9.1节。

5. 如果服务端选择接受一条连接，他必须发送一个如下说明的有效的HTTP请求来进行相应。
	1. 像[RFC2616][17]中说明的一样，状态码为101的状态行。比如看上去像这种的："HTTP/1.1 101 Switching Protocols"。
	2. 像[RFC2616][18]中说明的一样，值为"websocket"的`Upgrade`header字段。
	3. 值为"Upgrade"的`Connection`header字段。
	4. 一个`Sec-WebSocket-Accept`header字段。这个值由第4.2.2节的第4步提到的key来进行构造，通过和字符串"258EAFA5-E914-47DA-95CA-C5AB0DC85B11"拼接在一起进行SHA-1哈希运算，得到一个20字节的值，然后对这20字节进行base64编码。
	ABNF对这个字段定义如下：

		Sec-WebSocket-Accept = base64-value-non-empty
		base64-value-non-empty = (1*base64-data [ base64-padding ]) | base64-padding
		base64-data = 4base64-character
		base64-padding = (2base64-character "==") | (3base64-character "=")
		base64-character = ALPHA | DIGIT | "+" | "/"

	注意：作为示例，如果客户端握手时发送的`Sec-WebSocket-Key`header字段的值为"dGhlIHNhbXBsZSBub25jZQ=="，那么服务端会把"258EAFA5-E914-47DA-95CA-C5AB0DC85B11"拼接到后面得到"dGhlIHNhbXBsZSBub25jZQ==258EAFA5-E914-47DA-95CA-C5AB0DC85B11"。然后服务端回对这个字符串进行SHA-1哈希操作，得到0xb3 0x7a 0x4f 0x2c 0xc0 0x62 0x4f 0x16 0x90 0xf6 0x46 0x06 0xcf 0x38 0x59 0x45 0xb2 0xbe 0xc4 0xea。对这个值进行base64编码，得到结果为"s3pPLMBiTxaQ9kYGzzhZRbK+xOo="，然后通过`Sec-WebSocket-Accept`字段返回这个结果。
	
	5. 可选的`Sec-WebSocket-Protocol`字段，值为定义在第4.2.2节第4点中的子协议中。
	6. 可选的`Sec-WebSocket-Extensions`字段，值为定义在4.2.2节第4点中的扩展中。

这样服务端握手响应就完成了。如果服务端完成了上述步骤时也没有关闭中断WebSocket连接，那么服务端会考虑建立这个WebSocket连接并且将WebSocket连接状态置为`OPEN`。在此刻，服务端就可以开始发送（和接收）数据了。

## 4.3 收集握手中使用的新的ABNF的header字段

这一节使用在[RFC2616第2.1节][19]定义的ABNF语法和规则，包括隐含的*LWS规则（implied *LWS rule）。

请注意本节中使用了一下ABNF规定。一些规则名称对应一些header字段。这样的规则表示对应的header字段的值，例如`Sec-WebSocket-Key`的ABNF描述了`Sec-WebSocket-Key`header字段的值的语法。在名字中带有"-Client"后缀的ABNF规则只适用于客户端发送给服务端的请求；而名字中带有"-Server"后缀的ABNF规则则只适用于服务端给客户端发送的请求响应。例如ABNF规则`Sec-WebSocket-Protocol-Client`表示客户端发送给服务端的请求中的`Sec-WebSocket-Protocol`字段的值。

以下的新的header字段可以在客户端向服务端发送握手请求时使用：

	Sec-WebSocket-Key = base64-value-non-empty
	Sec-WebSocket-Extensions = extension-list
	Sec-WebSocket-Protocol-Client = 1#token
	Sec-WebSocket-Version-Client = version
	
	base64-value-non-empty = (1*base64-data [ base64-padding ]) | base64-padding
	base64-data = 4base64-character
	base64-padding = (2base64-character "==") | (3base64-character "=")
	base64-character = ALPHA | DIGIT | "+" | "/"
	extension-list = 1#extension
	extension = extension-token *( ";" extension-param )
	extension-token = registered-token
	registered-token = token
	extension-param = token [ "=" (token | quoted-string) ]
	    ;当使用带引号的字符串语法变体时，在引号转义后面的值必须和ABNF"标记（token）"一致。
	NZDIGIT =  "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9"
	version = DIGIT | (NZDIGIT DIGIT) | ("1" DIGIT DIGIT) | ("2" DIGIT DIGIT)
	    ; 范围是从0-255，没有前导0

以下的新的header字段可以在服务端向客户端发送握手响应请求时使用：

	Sec-WebSocket-Extensions = extension-list
	Sec-WebSocket-Accept = base64-value-non-empty
	Sec-WebSocket-Protocol-Server = token
	Sec-WebSocket-Version-Server = 1#version

## 4.4 支持多版本WebSocket协议

这一节提供了一些关于在客户端和服务端间支持多版本的WebSocket的协议的指导。

使用WebSocket版本标记字段（`Sec-WebSocket-Version`header字段），客户端可以在最初请求时选择WebSocket协议的版本号（客户端不必要支持最新的版本）。如果服务端支持请求的版本并且我收到消息是有效的，那么服务端会接受这个版本。如果服务端不支持客户端请求的版本，那么服务端必须返回一个`Sec-WebSocket-Version`header字段（或者多个`Sec-WebSocket-Version`header字段）包含服务端支持的所有版本。在这种情况下，如果客户端支持其中任意一个版本，它可以选择一个新的版本值重新发起握手请求。

下面的示例演示了如何进行上面所述的版本协商：

	GET /chat HTTP/1.1
	Host: server.example.com
	Upgrade: websocket
	Connection: Upgrade
	...
	Sec-WebSocket-Version: 25

服务端的响应可能如下所示：

	HTTP/1.1 400 Bad Request
	...
	Sec-WebSocket-Version: 13, 8, 7

注意服务端发送的最后的请求响应也可能是这个样子：

	HTTP/1.1 400 Bad Request
	...
	Sec-WebSocket-Version: 13
	Sec-WebSocket-Version: 8, 7

客户端选择了版本13，重新进行握手：

	GET /chat HTTP/1.1
	Host: server.example.com
	Upgrade: websocket
	Connection: Upgrade
	...
	Sec-WebSocket-Version: 13

[1]:	https://tools.ietf.org/html/rfc1928
[2]:	https://tools.ietf.org/html/rfc2616
[3]:	https://tools.ietf.org/html/rfc4648#section-4
[4]:	https://tools.ietf.org/html/rfc6454
[5]:	https://tools.ietf.org/html/rfc2616
[6]:	https://tools.ietf.org/html/rfc2616
[7]:	https://tools.ietf.org/html/rfc6265
[8]:	https://tools.ietf.org/html/rfc2616
[9]:	https://tools.ietf.org/html/rfc2616
[10]:	https://tools.ietf.org/html/rfc6265
[11]:	https://tools.ietf.org/html/rfc2616
[12]:	https://tools.ietf.org/html/rfc4648#section-4
[13]:	https://tools.ietf.org/html/rfc2616
[14]:	https://tools.ietf.org/html/rfc2616
[15]:	https://tools.ietf.org/html/rfc2616
[16]:	https://tools.ietf.org/html/rfc2616
[17]:	https://tools.ietf.org/html/rfc2616
[18]:	https://tools.ietf.org/html/rfc2616
[19]:	https://tools.ietf.org/html/rfc2616#section-2.1