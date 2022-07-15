# 概述

本文为WebSocket协议的第二章，本文翻译的主要内容为WebSocket协议中相关术语的介绍。


# 2 一致性要求（第二章协议正文）

在这篇文档中，所有的图、示例和笔记都是非规范性的，就像标注了非规范性的所有章节一样。在文档中没有指定的其他内容都是规范性的。

在这篇文档中的关键词如“必须（MUST）”、“必须不（MUST NOT）”、“需要（REQUIRED）”、“应该（SHALL）”、“不应该（SHALL NOT）”、“应该（SHOULD）”、“不应该（SHOULD NOT）”、“推荐（RECOMMENDED）”、“也许（MAY）”和“可选（OPTIONAL）”可以按照[RFC2119
](https://tools.ietf.org/html/rfc2119)所述进行解释。

作为算法的一部分的命令式语句（如“删除任何前导空格”或“返回false并且中止后续步骤”）在介绍算法时应该与关键词一起解释（“必须（MUST）”、“应该（SHOULD）”、“也许（MAY）”等）。

算法或者指定步骤中的符合要求的措辞可以通过任何方式表述，只要最终的结果是等价的。（尤其是在算法定义中，我们的目标是竟可能简单的操作而不是最求完美。）

## 2.1 术语和其他公约

\_ASCII\_表示定义在[ANSI.X3-4.1986][1]的字符编码表。

这个文档参考UTF-8的值，使用在STD 63（[RFX3629][2]）定义的UTF-8标准格式。

如命名算法或者定义关键输入的标识如\_this\_。

命名header字段或者变量如|this|。

本文引用了`WebSocket连接失败`（\_Fail the WebSocket Connection\_）这个程序。这个程序位于第7.1.7节。

`转换小写字符`（\_Converting a string to ASCII lowercase\_）意味着替换从U+0041到U+005A的所有字符（拉丁字母大写A到Z）为相对应的U+0061到U+007A的字符（拉丁字母小写A-Z）。

不区分ASCII大小写（\_ASCII case-insensitive\_）比较方式意味着通过码点（code point）比较这两个字符，如果这两个字符是U+0041到U+005A（拉丁字母大写A到Z）和相对应的U+0061到U+007A的字符（拉丁字母小写A-Z），那么也认为这两个字符相等。

文档中`URI`这个词被定sj义在了[RFC3986][3]。

当需要实现WebSocket协议中一部分的\_send\_数据时，这个实现是有可能会延迟任意时间来进行数据传输的，例如，使用数据缓冲区来保证发送较少的IP数据包。

这个文档在不同的章节会同时使用[RFC5234][4]和[RFC2616][5]这两个中的扩充巴科斯-瑙尔范式（ABNF）。


[1]:	https://tools.ietf.org/html/rfc6455#ref-ANSI.X3-4.1986
[2]:	https://tools.ietf.org/html/rfc3629
[3]:	https://tools.ietf.org/html/rfc3986
[4]:	https://tools.ietf.org/html/rfc5234
[5]:	https://tools.ietf.org/html/rfc2616
