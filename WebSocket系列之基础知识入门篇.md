# 概述

本文是WebSocket系列的第一篇，主要介绍WebSocket相关的基础协议知识和API。由于WebSocket的相关介绍在MDN中分布较乱，初学者不太容易入门，因此通过本文将相关基础知识和使用方法进行一个归纳和总结。

本文主要内容如下：

- WebSocket基础概念介绍
- WebSocket协议初读
- WebSocket 相关API浅析
- WebSocket在线上项目中的使用

通过本文，你能够了解到WebSocket相关基础知识，同时了解到WebSocket在线上环境中是如何使用的。

# WebSocket介绍

> **WebSockets** 是一个可以创建和服务器间进行双向会话的高级技术。通过这个API你可以向服务器发送消息并接受基于事件驱动的响应，这样就不用向服务器轮询获取数据了。

上面是[MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/WebSockets_API)中关于WebSocket的说明。其中`双向会话`指的是客户端和服务端都能够通过WebSocket来进行数据的互相传递，即服务端可以给客户端推送数据，客户端也可以通过WebSocket来传递数据。

## 为什么要使用WebSocket

在不使用WebSocket时，如果我们需要建立一条长连接，有以下几种方法：

- 轮询
- 长轮询（常用）
- SSE(Server Send Event)

下面，我们对这几个都进行简单的介绍。

### 轮询

轮询是最早在客户端用来模拟长连接的一种方式。他通过客户端定时想服务端发送HTTP请求来模拟客户端向服务端发送数据，而服务端的数据则是在客户端发送HTTP请求后跟随返回。

这种方案能够让客户端的数据几乎实时的到达，但是缺点也显而易见：服务端的数据需要在客户端的请求回来后才能带回。如果HTTP请求的间隔太短，则会导致大量的网络开销；如果间隔太长，这将导致数据传递的不及时。

### 长轮询

长轮询是在轮询的基础上改进的一种方式。在客户端发送HTTP请求且服务端收到请求时，服务端会先维持这个请求不返回。在特定的时间内（一般为30秒，因为通常HTTP判断超时时间为30秒），如果服务端没有数据，则回应这个请求；服务端有数据需要发送时，则立即通过HTTP请求的响应将数据传递给客户端。客户端收到响应后，立即发起下一次的HTTP请求。

这种方案能够解决轮询中带来的服务端数据不能及时传递的问题，但是带来的网络花销大的问题仍然无法解决。

### SSE(Server Send Event)

SSE是一个新的协议，作用为服务端想客户端推送数据。他通过自定义的SSE协议来实现单项的数据推送。SSE的缺点是数据只能从服务端像客户端传递，而数据不能通过客户端向服务端传递。

### WebSocket能够解决上述问题

WebSocket能够有效的解决以下问题：

1. 带宽问题：WebSocket相对于HTTP来说协议头更加小，同时按需传递。
2. 数据实时性问题：WebSocket相对于轮询和长轮询来说，能够实时传递数据，延迟更小。
3. 状态问题：相较于HTTP的无状态请求，WebSocket在建立连接后能够维持特定的状态。

其他的优点可以参考[维基百科](https://zh.wikipedia.org/wiki/WebSocket)。

# WebSocket协议

了解了为什么需要使用WebSocket，下面让我们来了解下WebSocket协议相关的内容。

WebSocket协议是通过HTTP协议升级而来。只需要在HTTP协议基础上增加两次握手，即可建立WebSocket连接（如果是需要通过SSL加密，则还需要进行SSL握手过程），握手的部分详情可以见[WebSocket文档](https://tools.ietf.org/html/rfc6455#section-1.3)，下面我们简单介绍以下Header相关字段。

## 请求Header

请求Header如下：

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

其中：

- `Host: server.example.com`：表示将要连接的WebSocket地址。
- `Connection: Upgrade`：需要升级HTTP连接。
- `Upgrade: websocket`：将HTTP连接升级至WebSocket连接。
- `Sec-WebSocket-Key:dGhlIHNhbXBsZSBub25jZQ== `：客户端生成的WebSocket连接密钥。
- `Sec-WebSocket-Protocol: chat, superchat`：指定哪些协议是客户端可以接受的。
- `Sec-WebSocket-Version: 13`：WebSocket版本号。

## 响应Header

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat
```

其中：

- `Upgrade: websocket`：确认将HTTP连接升级至WebSocket连接。
- `Connection: Upgrade`：确认升级HTTP连接。
- `Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo`：服务端根据客户端的连接密钥生成的服务端密钥。
- `Sec-WebSocket-Protocol: chat`：选择的WebSocket协议。

# WebSocket API介绍

对WebSocket的协议有了一个初步的了解，下面让我们看下，在具体的使用场景中，如何使用WebSocket。

WebSocket的API不多，下面我们就根据使用的顺序：

- 建立连接
- 收到消息
- 发送消息
- 关闭连接

来逐一进行介绍，具体的MDN资料可以见[此处](https://developer.mozilla.org/zh-CN/docs/Web/API/WebSocket)。

## 建立连接

WebSocket通过初始化实例来建立连接，通过`open`事件回调函数来确认连接建立成功，具体示例如下：

```javascript
const webSocket = new WebSocket('ws://server.example.com');

webSocket.addEventListener('open', (event) => {
    // 建立连接成功
});
```

在WebSocket建立ws连接时，url可以是域名或者IP地址；但是当建立的连接是wss（加密WebSocket）时，url必须是域名，因为需要配置相应的证书，而证书是针对域名的。

## 收到消息

WebSocket通过`message`事件来接收消息。

```javascript
socket.addEventListener('message', function (event) {
    console.log('Message from server', event.data);
});
```

WebSocket可以传递`String`、`ArrayBuffer`和`Blob`三种数据类型，因此在收到消息时可能是其中的任意一种。其中，`String`和`ArrayBuffer`使用的最多。

- 如果是`String`类型，直接通过字符串处理函数即可进行相关转换，如JSON等格式。


- 如果是二进制`ArrayBuffer`类型，则需要使用`DataView`来进行处理，相关的内容将在本系列第二篇中进行介绍。

## 发送消息

WebSocket通过`send`方法来发送消息。

```javascript
webSocket.send(data);
```

示例中的`data`字段，也有可能是收到消息所说的`String`、`ArrayBuffer`和`Blob`三种数据类型之一。其中，`Blob`作为一种类文件数据类型，再此不进行过多介绍。我们使用最多的就是`String`和`ArrayBuffer`。

- `String`类型只需要传递一个字符串给`send`方法作为参数即可。
- `ArrayBuffer`类型则需要传递一个ArrayBuffer对象作为参数，相关的内容也将在本系列第二篇中进行介绍。

## 关闭连接

### 被动关闭

当服务端主动关闭WebSocket连接时，会通过WebSocket向客户端发送一个close数据包，WebSocket的`close`事件会触发。

```javascript
webSocket.addEventListener('close', (closeEvent) => {
    
});
```

**注：当网络断开时，WebSocket连接并不会被动关闭，因为没有收到关闭的数据包。**

### 主动关闭

客户端可以通过WebSocket提供的`close`方法来主动关闭长连接。

```javascript
webSocket.close();
```

目前该方法有两个参数（在某些版本中不支持，详情见[MDN文档](https://developer.mozilla.org/zh-CN/docs/Web/API/WebSocket#close())）：

- 第一个参数表示关闭连接的状态号，默认为1000，表示正常关闭。
- 第二个参数为关闭原因，是一个不长于123字节的UTF-8文本。

**注：主动关闭不会触发`close`事件。**

# 总结

本文主要是介绍了一下WebSocket相关的基础知识。

通过WebSocket的长连接，客户端和服务端可以进行大量的数据传输而不会带来相关的性能问题，这给Web端带来了极大的功能增强。目前Web端可以使用WebSocket来进行IM相关功能开发，或者实时协作等需要与服务端进行大量数据交互的功能，并且不需要像之前一样使用长轮询的Hack方式来实现。

具体的使用方式和线上的使用问题，将会在本系列后几篇博客中进行介绍。