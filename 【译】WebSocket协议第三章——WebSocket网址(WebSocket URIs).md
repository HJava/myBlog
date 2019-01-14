# 概述

本文为WebSocket协议的第三章，本文翻译的主要内容为WebSocket连接的相关URI地址介绍。

# WebSocket URIs（第三章协议正文）

这个规范使用在[RFC5234][1]中的ABNF语法以及URI规范中的[RFC3986][2]的术语和ABNF产品定义了两套方案。

	ws-URI = "ws:" "//" host [ ":" port ] path [ "?" query ]
	wss-URI = "wss:" "//" host [ ":" port ] path [ "?" query ]
	
	host = <host, defined in [RFC3986](https://tools.ietf.org/html/rfc3986#section-3.2.2), 3.2.2节>
	port = <port, defined in [RFC3986](https://tools.ietf.org/html/rfc3986#section-3.2.3), 3.2.3节>
	path = <path-abempty, defined in [RFC3986](https://tools.ietf.org/html/rfc3986#section-3.3), 3.3节>
	query = <query, defined in [RFC3986](https://tools.ietf.org/html/rfc3986#section-3.4), 3.4节>

端口字段是可选的，默认的"ws"端口是80，而默认的"wss"端口是443。

命中不论大小写的"wss"方案字段就表明这个URI可以被称为安全的（已经设置安全标记）。

"resource-name"字段（在4.1节也被称为/resouce name/字段）可以通过以下的数据串联起来：

- "/"，表示路径（path）字段为空
- 路径（path）字段
- "?"，表示非空的查询参数（query）
- 空查询参数（query）

在WebSocket URIs的里，身份标识片段是没有意义的，而且禁止使用在这些URI里面。与任何的URI方案一样，"#"字符不是表示片段（fragment）开始时，都必须编码为`%23`。

[1]:	https://tools.ietf.org/html/rfc5234
[2]:	https://tools.ietf.org/html/rfc3986