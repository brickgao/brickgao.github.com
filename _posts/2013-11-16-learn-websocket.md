---
layout: post
title: Websocket初探
disqus: y
date: 2013-11-16 14:23:05
tags: [Node.js, websocket, socket.io]
---

![> <!](/media/2013/11/Komeiji_Satori.jpg)

一时兴起想写一个实时的弹幕墙，本来想着使用 socket 协议来写，服务端两个模块一个做 socket server ，一个做 web 服务器，客户端两个模块，一个做 socket server ，一个 GUI 程序来显示弹幕， web 服务器接到弹幕的信息就发送给控制 socket 的部分， socket server 发送信息给 socket client ，然后将 socket client 接收到的信息传送给 GUI 程序。

这么来说一下是要写四个模块，感觉非常麻烦，于是就想着能不能单纯用web来实现这个功能，后来[杭菊苣](http://mad4a.me/ "OMG杭菊苣")给我说可以试试看 comet ， wiki 了一下发现 comet 是轮询操作，太烧服务器了，后来在下面的页面里发现了 websocket 的存在，于是就用 websocket 开坑了。

webSocket 是 HTML5 开始提供的一种浏览器与服务器间进行全双工通讯的网络技术，简单里说就是 web 端的socket，在 WebSocket API 中，浏览器和服务器只需要要做一个握手的动作，然后，浏览器和服务器之间就形成了一条快速通道。两者之间就直接可以数据互相传送。

于是刚才四个模块的四个模块就被全部做到 web 端了，模型如下：

![通信模型](/media/2013/11/websocket.png)

在这里我使用是 socket.io 模块，尽管严格意义上说 socket.io 应该算是 websocket 的拓展。

具体使用 socket.io 的方法如下：

在`package.json`中添加`"socket.io": "*"`，然后`npm install`。

在客户端中，添加`<script src="/socket.io/socket.io.js"></script>`，引入 socket.io。

在服务端的`app.js`中，添加`var io = require('socket.io').listen(server);`，其中`server`是 `http.createServer(app).listen(app.get('post'))` 创建的服务器监听对象，然后再用 `io.sockets.on('connection', function() {})` 表示握手，后面的回调就写 websocket 的操作。

这里介绍一下 socket.io 的两个具体方法，一个是`on`，是对接受的信息的处理，一个是`emit`，是对消息的广播。具体的用法请参见[ socket.io 的文档](https://github.com/learnboost/socket.io/tree/master "socket.io 的文档")。

当我们执行`node app.js`之后可以看到

{% highlight sh %}
$ node app.js
   info  - socket.io started
Express server listening on port 3000
GET / 200 22ms - 3.08kb
GET /stylesheets/base.css 304 5ms
   debug - served static content /socket.io.js
GET /javascripts/post.js 304 1ms
   debug - client authorized
   info  - handshake authorized 32EwuoTGmPKqoBp-E7SQ
   debug - setting request GET /socket.io/1/websocket/32EwuoTGmPKqoBp-E7SQ
   debug - set heartbeat interval for client 32EwuoTGmPKqoBp-E7SQ
   debug - client authorized for
   debug - websocket writing 1::
   debug - emitting heartbeat for client 32EwuoTGmPKqoBp-E7SQ
   debug - websocket writing 2::
   debug - set heartbeat timeout for client 32EwuoTGmPKqoBp-E7SQ
   debug - got heartbeat packet
   debug - cleared heartbeat timeout for client 32EwuoTGmPKqoBp-E7SQ
   debug - set heartbeat interval for client 32EwuoTGmPKqoBp-E7SQ
   debug - emitting heartbeat for client 32EwuoTGmPKqoBp-E7SQ
   debug - websocket writing 2::
   debug - set heartbeat timeout for client 32EwuoTGmPKqoBp-E7SQ
   debug - got heartbeat packet
   debug - cleared heartbeat timeout for client 32EwuoTGmPKqoBp-E7SQ
   debug - set heartbeat interval for client 32EwuoTGmPKqoBp-E7SQ
{% endhighlight %}

这里`socket.io`把每一个窗口视为了一个`client`，并且分配了唯一不变的编号，在一次握手之后建立了连接，之后没隔一段时间通信一次确定连接是否断开（这里叫heartbeat，很有意思）。

当我们广播信息的时候，我们可以看到：

{% highlight sh %}
   debug - websocket writing 5:::{"name":"msg","args":[{"msg":"sss","color":"#000000","size":"40px"}]}
{% endhighlight %}

即我们广播的某条信息的内容。

可以说 websocket 改善了 web 端 server 和 client 之间的交互，相对于长轮询消耗要小，虽然维护长连接也需要一定的成本，或许随着技术的完善， websocket 会更加普及。
