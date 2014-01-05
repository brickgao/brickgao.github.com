---
layout: post
title: 从 socket 开始实现一个简单的 FTP 客户端
disqus: y
date: 2014-01-05 23:27:42
tags: [Python, socket, FTP]
---

之所以从 socket 开始写 FTP ，完全是因为这是计算机网络的大作业，所以姑且借着这个机会研究一下 FTP。

### FTP 相关的概念

文件传输协议（英文：File Transfer Protocol，缩写：FTP）是用于在网络上进行文件传输的一套标准协议。它属于网络传输协议的应用层。从个人理解上说， FTP 就是一个规范化的 socket ，用于 client 和 server 之间文件传输。

一般来说， FTP 运行于 20/21 端口 ，20 端口用于数据流，也就是上传/下载数据，21 端口用于控制流，通常有两种连接方式，一种是主动模式，另一种是被动模式，主动模式即是通过 20/21 端口实现 FTP，而被动模式则是服务器新开一个随机的端口用于数据流，以避免服务器防火墙的问题。这里我们主要实现的是被动模式，其实主动模式实现方式也差不多。

### 通过 socket 来实现 FTP

如果对 FTP 的相关标准感兴趣，在看这个之前可以参照一下 [RFC 0959](http://tools.ietf.org/html/rfc0959) 。

实现 FTP 的首先当然是要连接服务器，我们通过 socket 简单连接即可。

{% highlight python %}
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect((self.url, 21))
{% endhighlight %}

连接成功之后我们可以接受到服务器发来的信息，这个信息包含了状态号以及服务端的名称和版本号。

例如我这里连接`ftp://ftp.sjtu.edu.cn/`，就会收到：

{% highlight bash %}
220 (vsFTPd 2.2.2)
{% endhighlight %}

这里 220 代表 Service ready for new user，就是服务端已经准备好对一个新用户开放了。

从这之后我们就开始以`command+\r\n`的形式来发送命令（你收到的服务器的消息也是`msg+\r\n`形式的），这里列举一些常用的命令，更多的命令可以查看 [RFC 0959](http://tools.ietf.org/html/rfc0959) 。

#### USER 和 PASS

这两个命令用于登录，意思就是字面意思。

这里以匿名登录为例。

{% highlight python %}
sock.sendall('USER anonymous\r\n')
_ = sock.recv(1024)
print _
{% endhighlight %}

用户名发送成功后，会以 331 反馈， 331 即 User name okay, need password。

{% highlight python %}
sock.send('PASS \r\n')
_ = sock.recv(1024)
print _
{% endhighlight %}

这里匿名帐号是没有密码的，如果密码正确，会以 230 反馈， 230 即 User logged in, proceed，收到了 230 ，就去做你想做的事情吧。

**TIPS**: 匿名用户一般用户名为 anonymous ，密码为空，如果有一些服务端不是这个，你可以根据之前服务端反馈的名称判断。

#### PASV

PASV 是进入被动模式的命令，当你发送`PASV\r\n`之后，会收到 227 的状态码，以及6个数字：

{% highlight bash %}
227 Entering Passive Mode (202,38,97,230,254,135).
{% endhighlight %}

(h1, h2, h3, h4, p1, p2) 这六个数字的前四个代表被动模式连接的 ip ，这里的例子为`202.38.97.230`，后面两个数字代表端口号，具体为`p1 * 256 + p2` ，这里就是 254 * 256 + 135 = 65159。

这之后我们新开一个 socket 对象来连接这个 ip 的端口，用于数据流，而原来的 socket 保留用来做控制流。

**TIP**: 被动模式的这个 socket 会在每次使用完数据流的时候失效，所以可以选择在每次使用数据流之前再申请被动模式的端口。（我这里没有写主动模式，听[杭神](mad4a.com)说主动模式也是数据流用完，数据流端口就会失效。）

#### CWD

CWD 是切换目录的命令，发送时发送`'CWD ' + 目录 + '\r\n'`，切换成功后从控制流端口返回 226 状态码。

#### LIST

LIST 是列出目录下所有文件信息的命令，发送时发送`LIST\r\n`，如果顺利的话，会先从控制流端口返回 150 状态码，然后从数据流端口开始传入数据，传完后以控制流端口返回 226 结束。

这里 [RFC 0959](http://tools.ietf.org/html/rfc0959) 并没有规定 LIST 的返回格式，然后看`libftp`也是直接返回 raw 的格式，但是一般来说基本是`ls -la`的返回格式，每个文件以`\r\n`分割，如果你是 LINUX 用户，我相信你应该很容易看懂，如果是其他情况的话，也就只能对具体服务器具体操作了。

具体我这里是对收到的数据用`split('\r\n')`来得到一个 list ，之后对 list 中的每一个元素，用正则筛选，最后得到一个 dict ，以便 GUI 的操作。

#### SIZE

获得文件的大小，具体命令是发送`'SIZE ' + 文件名 + '\r\n'`，文件如果存在的话，返回`213 <size>`。

#### RETR

下载文件，具体命令是发送`'RETR ' + 文件名 + '\r\n'`，如果文件存在，则先从控制流端口返回 150 ，然后从数据流端口开始发送文件，我们因为知道了文件的大小，用 socket 来接收数据的时候就会很好处理，发送完成从控制流端口后返回 226 。

#### STOR

上传文件，基本与下载的格式一致，具体命令是发送`'STOR ' + 文件名 + '\r\n'`，依然是以 150 开始接受，发送完成后从控制流端口返回 226 。

#### QUIT

关闭连接，具体为发送`QUIT\r\n`，收到 221 为成功关闭。

### GUI 部分

完成了 FTP 的部分，我们就可以开始写 GUI 部分了，GUI 部分其实并没有什么好说的，这里我用的是 PyQt4，我用`logging`在 FTP 模块里生成日志，然后把他的`handle`加到 GUI 的`QTextBrowser`中，以实现 FTP 客户端的日志功能，`QTextBrowser`支持简单的 html 渲染，所以你可以把 logger 做成彩色的，具体可以参见[在PyQt中实现一个可以变色的log窗口](http://mad4a.me/2013/05/13/log-handler-with-qtextbrowser/)这篇文章，除了注意`QString`类型与 Python 一般的`unicode`和`str`类型的转换外，请注意线程安全，在 PyQt4 中，QWidget 并不是现成安全的，之前把列表刷新操作写在了线程里，结果出现了经典的内存不可读写错误，之后用信号量解决了，在 PyQt4 中，信号量是线程安全的。

### 最后

这里实现了一个不完整的 FTP 库，不过不是那么漂亮，至于更优美的解决方案，建议去看一下`libftp`，学习归学习，对于已经有了还好的轮子的东西，我并不建议自己造轮子。

至于具体完成的代码，可以参见[这里](https://github.com/brickgao/SimpleFTP)。

拓展阅读：
[RFC 0959](http://tools.ietf.org/html/rfc0959) - FTP 的标准  
[使用 Socket 通信实现 FTP 客户端程序](http://www.ibm.com/developerworks/cn/linux/l-cn-socketftp/) - IBM developerWorks 中国

