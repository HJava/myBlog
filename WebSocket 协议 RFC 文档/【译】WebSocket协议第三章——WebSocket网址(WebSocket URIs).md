# 概述

本文为WebSocket协议的第三章，本文翻译的主要内容为WebSocket连接的相关URI地址介绍。

# WebSocket URIs（第三章协议正文）

这个规范使用在[RFC5234][1]中的ABNF语法以及URI规范[RFC3986][2]中的术语和ABNF产生式定义了两种URI方案（scheme）。

	ws-URI = "ws:" "//" host [ ":" port ] path [ "?" query ]
	wss-URI = "wss:" "//" host [ ":" port ] path [ "?" query ]
	
	host = <host, defined in [RFC3986](https://tools.ietf.org/html/rfc3986#section-3.2.2), 3.2.2节>
	port = <port, defined in [RFC3986](https://tools.ietf.org/html/rfc3986#section-3.2.3), 3.2.3节>
	path = <path-abempty, defined in [RFC3986](https://tools.ietf.org/html/rfc3986#section-3.3), 3.3节>
	query = <query, defined in [RFC3986](https://tools.ietf.org/html/rfc3986#section-3.4), 3.4节>

端口字段是可选的，默认的"ws"端口是80，而默认的"wss"端口是443。

如果方案（scheme）字段不区分大小写地匹配"wss"，那么这个URI就被称为安全的（即已经设置安全标记）。

"resource-name"字段（在4.1节也被称为/resource name/字段）可以通过以下的数据串联起来：

- "/"，如果路径（path）字段为空
- 路径（path）字段
- "?"，如果查询参数（query）非空
- 查询参数（query）字段

在WebSocket URI里，片段标识符（fragment identifier）是没有意义的，而且禁止使用在这些URI里面。与任何的URI方案一样，当"#"字符不是用来表示片段（fragment）的开始时，都必须编码为`%23`。

[1]:	https://tools.ietf.org/html/rfc5234
[2]:	https://tools.ietf.org/html/rfc3986