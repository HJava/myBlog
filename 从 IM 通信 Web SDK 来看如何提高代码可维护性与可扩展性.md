
## 本文内容概述

在架构设计和功能开发中，代码的可维护性和可扩展性一直是工程师不懈的追求。本文将以我工作中开发的 IM 通信服务 SDK 作为示例，和大家一起探讨下前端基础服务类业务的代码中对可维护性和可扩展方面的探索。

本文不涉及具体的代码和技术相关细节，如果想了解 IM 长连接相关的技术细节，可以阅读我之前的文章：

- [WebSocket系列之基础知识入门篇][1]
- [WebSocket系列之JavaScript数字数据如何转换为二进制数据][2]
- [WebSocket系列之JavaScript字符串如何与二进制数据间进行互相转换][3]
- [WebSocket系列之二进制数据设计与传输][4]
- [WebSocket系列之如何建立和维护可靠的连接][5]

## 背景介绍

大象 SDK 是美团生态中负责 IM 通信服务的基础服务。作为 IM 通信服务的 Web 端载体，我们对不同的业务线提供不同的功能来满足特定的需求，同时需要支持 PC、Web、移动端H5、微信小程序等各个平台。

不同的业务方需求和不同的平台对 Web SDK 的功能和模块要求都不相同，因此在整个 Web SDK 中有许多部分存在需要适配多场景的情况。

处理这种常见的场景，我们一般有以下几个思路：

1. 针对不同的场景单独开发不同的基础服务代码。这种操作灵活性最强，但是成本也是最高的，如果我们需要面对 M 个业务需求和 N 个平台，我们就需要有 M \* N 套代码。这个对于技术人员来说，基本上是一个不可能接受的情况。
2. 将所有的代码全部聚合到一个业务模块中，通过内部的 IF ELSE 判断逻辑来自动选择需要执行的代码逻辑。这种方案不会出现相同代码重复编写的情况，同时也兼顾了灵活性，看上去是一个不错的选择。但是我们仔细一想就会发现，所有的代码都堆积到一起，在后期会遇到大量的判断逻辑，在可维护性上来看是一个巨大的灾难。同时，我们所有的代码都放到一起，这会导致我们的包体积越来越大，而其他业务在使用相关功能时，也会引入大量无用代码，浪费流量。

那么，我们在既需要兼顾可维护性，有需要保证开发效率的情况下，我们应该如何去进行相关业务的架构设计呢？

## 核心原则

在我的设计理念中，有这么几个原则需要遵守：

1. 针对接口规范编程，而不针对特定代码编程（即设计模式中的策略模式）。我们在进行架构设计时，优先判断各个功能和模块中流转的数据格式和交互的数据接口规范，这样我们可以保证在进行特定代码编写的时候，只针对具体格式进行数据处理，而不会设计到数据内容本身。
2. 各模块权责分明，宽进严出。每个模块都是单一全责，暴露特定数据格式的 API，处理约定好数据格式的内容。
3. 提供方案供用户选择，而不帮用户做决策。我们不去判断用户所在环境、选择功能，而是提供多个选择来让用户主动去做这个决策。

## 具体实践

上面的原则可能比较抽象，我们来看几个具体的场景，大家就能够对这个有一个特定的概念。

### 连接模块设计（长连接部分）

连接模块包含长连接和短连接部分，我们在这里就用长连接部分来进行举例，短连接部分我们可以按照类似的原则进行设计即可。在设计长连接部分时，我们需要考虑的是：连接策略与切换策略。总的来说就是我们需要在什么时候使用哪一种长连接。

首先，我们以浏览器端为例，我们可以选择的长连接有：WebSocket 和长轮询。这个时候，我们可能首先以 WebSocket 优先，而长轮询作为备选方案来构成我们的长连接部分。因此，我们可能会在代码中直接用代码来实现这个方案。相关伪代码如下：

```ts
import WebSocket from 'websocket';
import LongPolling from 'longPolling';

class Connection {
	private _websocket;
	private _longPolling;

	constructor() {
		this._websocket = new WebSocket();
		this._longPollong = new LongPolling();
	}

	connect() {
		this.websocket.connect();
		// 只表达相关含义用于说明
		if (websocket.isConnected) {
			this.websocket.send(message);
		} else {
			this.longPolling.connect();
		}
	}
}
```

在正常情况下来看，我们发现这个代码没有什么问题。但是，如果我们的需求发生了某些变化呢？比如我们现在需要在某些特定的场景下，只开启长轮询，而不开启 WebSocket 呢（比如在 IE 浏览器里面）？之前的做法是在构造器的时候，传递一个参数进去，用来控制我们是不是开启 WebSocket。因此，我们的代码会变成以下的样子。

```ts
class Connection {
	private _useWebSocket;
	private _websocket;
	private _longPolling;

	constructor({useWebSocket}) {
		this._useWebSocket = useWebSocket;
		this._websocket = new WebSocket();
		this._longPollong = new LongPolling();
	}

	connect() {
		if (this._useWebSocket) {
			this.websocket.connect();
			// 只表达相关含义用于说明
			if (websocket.isConnected) {
				this.websocket.send(message);
			} else {
				this.longPolling.connect();
			}
		} else {
			this._longPolling.connect();
		}
	}
}
```

现在，我们通过增加一个判断参数，对`connect`函数进行了简单的改造，满足了在特定场景下的指使用长轮询的需求。

很不幸，我们的问题又来了，我们在针对移动端 H5 的场景下，我们需要一个只要 WebSocket 连接，而不需要长轮询。那么，根据我们之前的方式，我们可能又需要在增加一个新的参数`useLongPolling`。这个代码示例我就不增加了，大家应该能够想象出来。

在线上运行了一段时间后，新的需求又来了，我们需要在微信小程序里面支持 IM 的长连接。那么，根据我们之前的思路，我们需要在私有属性和connect方法中增加一堆判断逻辑。具体示例如下：

```ts
import WebSocket from 'websocket';
import LongPolling from 'longPolling';
import WXWebSocket from 'wxwebsocket';

class Connection {
	private _websocket;
	private _longPolling;
	private _wxwebsocket;

	constructor() {
		// 如果在微信小程序容器中
		if (isInWX()) {
			this._wxwebsocket = new WXWebSocket();
		} else {
			this._websocket = new WebSocket();
			this._longPollong = new LongPolling();
		}
	}

	connect() {
		if (isInWx()) {
			this._wxwebsocket.connect();
		} else {
			this.websocket.connect();
			// 只表达相关含义用于说明
			if (websocket.isConnected) {
				this.websocket.send(message);
			} else {
				this.longPolling.connect();
			}
		}
	}
}
```

从这个例子，大家应该可以发现相关的问题了，如果我们再支持百度小程序、头条小程序等更多的平台，我们就会在我们的判断逻辑里面加更多的逻辑，这样会让我们的可维护性有明显的下降。

现在有一些类库可以支持多平台的接口统一（大家去GitHub上面找一下就可以发现），那么为什么我没有用相关的产品呢？这是因为 SDK 作为一个基础服务，对包大小比较敏感，同时用到的需要兼容 API 并不多，所以我们自己做相关的兼容比较合适。

那么，我们应该如何设计这个方案，从而解决这个问题呢。让我们回顾下我们的设计理念。

1. 针对接口规范编程，而不针对特定代码编程。
2. 各模块权责分明，宽进严出。
3. 提供方案供用户选择，而不帮用户做决策。

通过这些设计理念，我们来看下具体的做法。

三个设计理念我们需要组合使用。首先是`针对结构规范编程`。我们来看下具体的用法。

首先我们定义一个长连接的接口如下：

```ts
export default interface SocketInterface {
    connect(url: string): void;
    disconnect(): void;
    send(data: any[]): void;
    onOpen(func): void;
    onMessage(func): void;
    onClose(func): void;
    onError(func): void;
	isConnected(): boolean;
}
```

有了这个长连接的接口类型后，我们可以让 WebSocket 和长轮询两个模块都实现这个接口。因此，他们就有了统一的 API。有了统一的 API 之后，我们就可以将连接策略中的操作“泛化”，从操作具体的连接方式转换为操作被选中的连接方式。

其次，根据我们的`各模块全责分明`的原则，我们的连接模块应该只控制我们的连接策略，并不需要关心她使用的是 WebSocket 还是长轮询，还是说微信小程序的 API。

道理很简单，但是具体我们应该怎么来实践呢？我们来看下下面这个示例：

```ts
class Connection {
	private _sockets = [];
	private _currentSocket;

	constructor({Sockets}) {
		for (let Socket of Sockets) {
			let socket = new Socket();
			socket.onOpen(() => {
				for (let socket of this._sockets) {
					if (socket.isconnected()) {
						this._currentSocket = socket;
					} else {
						socket.disconnect();
					}
				}
			});
			this._sockets.push(socket);
		}
	}

	connect() {
		for (let socekt of this._sockets) {
			socket.connect();
		}
	}
}
```

通过上面这个示例大家可以看到，我们泛化了每一个连接方式的差异，转为用统一的接口规范来约束相关的模块。这样带来的好处是，我们如果需要兼容 WebSocket 和长轮询时，我们可以把这两个的构造函数传递进来；如果我们需要支持微信小程序，我们也只需要将微信小程序的 API 封装一次，我们就可以得到我们需要的模块，这样可以保证我们的连接模块只负责连接，而不去关心它不该关心的兼容性问题。

那么由用户就会问了，那我们是在哪一层来判断传入的参数到底是哪些呢？是在这个模块的上一层吗？这个问题很简单，还记得我们的第三个规则是什么吗？那就是`提供方案供用户选择，而不帮用户做决策`。因此，我们在构建长连接部分的时候，我们就在 Webpack 里面定义一些常量用于判断我们当前构建时，我们生产的的包是用于什么场景。具体示例如下：

```ts
import Connection from 'connection';
import WebSocket from 'websocket';
import LongPolling from 'longPolling';
import WXWebSocket from 'wxwebsocket';

class WebSDK {
	private _connection;

	constructor() {
		if (CONTAINER_NAME === 'WX') {
			this._connection = new Connection({Sockets: [WXWebSocket]});
		}

		if (CONTAINER_NAME === 'PC') {
			this._connection = new Connection({Sockets: [WebSocket, LongPolling]});
		}

		if (CONTAINER_NAME === 'H5') {
			this._connection = new Connection({Sockets: [WebSocket]});
		}
	}
}
```

我们通过在 Webpack 中定义 `CONTAINER_NAME` 这个常量，我们可以在打包时构建不同的 Web SDK 包。在保证对外暴露 API 完全一致的情况下，业务方可以在不同的容器内，采用对应的打包方式，引入不同的 Web SDK 的包，同时不需要改动任何代码。

可能有人会问了，这个方式看上去其实和之前的方式没有什么不同，只是把这个 IF ELSE 的逻辑移动到了外面。但是，我可以告诉大家，这里有两个明显的优势：

1. 我们可以抽象单独的模块去管理和维护这个独立的判断逻辑，它不会和我们的长连接部分代码进行耦合。
2. 我们可以在打包过程中使用 tree-shaking，这样我们可以让我们的 Web SDK 构建的包中，不会出现我们不需要的模块的代码。

### 消息流处理

上面的长连接部分，我们看到了三个原则的使用。接下来我们来看下我们如何使用这个原则进行数据流的处理。

在 IM 场景中，我们会遇到许多类型的消息。我们以微信公众号为例，我们会碰到单聊（单人-单人）、群聊（单人-群组）、公众号（单人-公众号）等聊天场景。如果我们需要去计算消息的未读数，同时用消息来更新左侧的会话列表，我们就需要三套几乎完全一样的逻辑。

那么，我们有没有什么更优的方法呢。很明显，我们可以根据上面介绍的原则，定义一个消息接口。

```ts
interface MessageInterface {
	public fromId: string;
	public toId: string;
	public fromName: string;
	public messageType: number;
	public messageBody;
	public uuid: string;
	public serverId: string;
	public extension: string;
}
```

通过之前的例子，大家应该可以理解，我们现在的所有业务逻辑，比如更新未读数、更新会话列表的预览消息时，我们就只需要针对整个消息接口里面的数据进行处理。这样的话，我们的处理流程就会变成一个流水线作业，我们只负责处理特定逻辑的数据，而不管具体的数据内容是什么样子的。

因此，如果我们新增一类会话类型，比如客服消息，我们也可以按照上面这个接口去实现客服消息类，复用原来的逻辑，而不需要重新实现一套完整的代码。

我们的在一开始就需要对数据进行转换，这样才能够保证我们在内部流转时不会犹豫数据格式不同导致代码维护性变差。需要注意的是，根据我们的`各模块权责分明，宽进严出`原则，我们在像其他模块输出时，我们也需要保证我们只输出这一种格式的数据，而接受的数据，我们应该尽最大的努力去适应各种场景。

可能有人会问，我们内部自己规定使用那个系统就可以，控制了严出了，我们自然就不用处理宽进了。但是，你写的代码和模块很有可能会和其他人一起维护，这个时候，你只能从规范上面来约束他，而不能控制他。因此，我们在接收其他非同一开发模块的数据时，我们可能会遇到一些异常情况。这个时候如果我们对宽进有做处理，也能够保证该模块可以正常运行。

有了之前的经验，大家对这个示例应该很好理解，我就不多做介绍了。

## 总结

这一篇文章没有介绍什么代码层面的东西，而是和大家一起交流了一下，我在日常工作中遇到的一些可能的问题，以及关于设计模式相关的应用场景。

如果我们需要作为一个基础服务提供方，需要让自己的代码有扩展性和可维护性，我们需要：

1. 面对接口规范编程。
2. 单一全责、宽进严出。
3. 不帮用户做决策。

当然，在用户产品层面，可能上面的设计有部分相同的地方，也有部分不同的地方，有时间的话，我会在后面再和大家进行分享。

大家如果有兴趣的话可以在评论区发表下自己观点，也可以在评论里面留言进行讨论，也欢迎大家发表自己的观点。

## 作者介绍与转载声明

黄珏，2015年毕业于华中科技大学，目前任职于美团基础研发平台大象业务部，独立负责大象 Web SDK 的开发与维护。

本文未经作者允许，禁止转载。

[1]:	https://juejin.im/post/5ab91ac96fb9a028db58b1d5
[2]:	https://juejin.im/post/5abb560a6fb9a028d141262b
[3]:	https://juejin.im/post/5abdc38ef265da2375070008
[4]:	https://juejin.im/post/5abf5394f265da239e4e3545
[5]:	https://juejin.im/post/5ac5e8b06fb9a028b617beff