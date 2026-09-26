# 概述

本系列内容为[RFC6455 WebSocket协议][1]的中文翻译版。进行相关文档规范的翻译初衷是为了更加深刻地了解WebSocket以及相关内容。

本文主要为WebSocket协议

- 摘要

文章具体内容较少，后续会陆续更新相关的章节，有兴趣的同学可以持续关注一下。

翻译版包含了部分个人的理解，大部分内容为直译，其他小部分内容可能为意译，适合有兴趣的同学进行了解和学习。如果希望对整个WebSocket协议有具体的了解，建议对照英文文档进行阅读。如果有翻译上的错误，也欢迎大家指出。

PS：由于手骨折做手术导致博客停更了一周。目前已经出院，将恢复每周更新的频率。

# 摘要

WebSocket协议使得在受控环境中运行不可信代码的客户端，能够与已选择接受该代码通信的远端主机进行双向通信。这个用于WebSocket的安全模型是复用Web浏览器使用的基于Origin的安全模型（origin-based security model，可以参考[此处][2]）。这个协议由一个开始握手过程以及随后基于TCP的基本消息帧组成。这个技术的目标是提供基于浏览器的应用与服务端进行双向通信的机制，而不需要通过多个HTTP连接（例如使用XMLHttpRequest或者Iframe模拟长轮询）。

# 备忘录状态

这是一个互联网标准跟踪文档。

这个文档是由互联网工程任务组（IETF，Internet Engineering Task Force）产出的。它代表了互联网工程任务组社区的共识。这个文档已经征求过公众的意见并且互联网工程指导小组（IESG，Internet Engineering Steering Group）已经同意发布。更多关于互联网标准的信息在[RFC 5741的第二节][3]可以看到。

关于这篇文档当前状态的信息和勘误表，以及如何进行反馈可以在[此处][4]查看。

# 版权声明

版权所有（c）2011 IETF信托（IETF Trust）和被认定为文档作者的人。保留所有权利。

这个文档受[BCP 78][5]以及在此文档发布之日生效的IETF信托关于[IETF文档][6]的法律条款约束。请细心阅读这些文档，它们说明了对于这篇文档你的权利和限制。从此文档中提取的代码组件必须包含如信托法律条款第4.e节所述的简化BSD许可协议文本，并且如简化BSD许可协议所述，不提供任何担保。

[1]:	https://tools.ietf.org/html/rfc6455
[2]:	https://stackoverflow.com/questions/19148689/what-does-origin-based-security-model-mean?rq=1
[3]:	https://tools.ietf.org/html/rfc5741#section-2
[4]:	https://tools.ietf.org/html/rfc5741#section-2
[5]:	https://tools.ietf.org/html/bcp78
[6]:	https://tools.ietf.org/html/bcp78