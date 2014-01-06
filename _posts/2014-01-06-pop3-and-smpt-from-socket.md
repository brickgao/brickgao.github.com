---
layout: post
title: 从 socket 开始实现一个简单的邮件客户端
disqus: y
date: 2014-01-06 20:43:45
tags: [Python, socket, SMPT, POP3]
---

这次是从 socket 开始实现一个邮件客户端，也是计算机网络实验的作业之一，大概算是[从 socket 开始实现一个简单的 FTP 客户端](http://blog.brickgao.com/2014/01/05/ftp-from-socket.html)的姊妹篇了。

### MIME, SMTP 和 POP3 相关概念

电子邮件的概念就不再提了，相信各位都不陌生，这里要提的是 MIME ， MIME(ultipurpose Internet Mail Extensions) 是最初电子邮件设定的一个拓展，这个拓展允许了非 ASCII 字符、二进制格式附件等多种格式的邮件消息，可以说 MIME 是现代电子邮件中很重要的一环，具体的标准非常多，从[RFC 1847](http://tools.ietf.org/html/rfc1847)、[RFC 2045](http://tools.ietf.org/html/rfc2045)、[RFC 2046](http://tools.ietf.org/html/rfc2046)、[RFC 2047](http://tools.ietf.org/html/rfc2047)、[RFC 4288](http://tools.ietf.org/html/rfc4288)、[RFC 4289](http://tools.ietf.org/html/rfc4289)、[RFC 2049](http://tools.ietf.org/html/rfc2049)、[RFC 2231](http://tools.ietf.org/html/rfc2231)、[RFC 2387](http://tools.ietf.org/html/rfc2387)中都有提。

具体来说，作为一个 MIME 格式的邮件，一般包含有 From（发信人），To（收信人），Subject（邮件主题），Date（日期），MIME-Version（遵循 MIME 的版本），Content-Type（内容类型），Content-Transfer-Encoding（基本传输编码）等基本信息，你可以在自己手动编写的时候直接把 raw 的邮件输出看一下，然后可以选择自己处理或者直接用 Python 里 `email` 库里的东西。如果你想详细地了解邮件的格式信息，可以翻一下上面的标准。

再说说 SMTP(简单邮件传输协议，Simple Mail Transfer Protocol) 和 POP(邮局协议，Post Office Protocol)，POP3 是 POP 的最新标准，简单点来说，SMTP 是发送邮件相关的协议，而 POP3 是管理收件相关的协议，当然管理收件我们也可以选择 IMAP ，IMAP 相对于 POP3 更加灵活一些，如果感兴趣 IMAP 、 SMTP 以及 POP 的标准，可以 wiki 一下，这里不再多说。

### 实现 POP3

个人感觉无论是 POP3 还是 SMTP 来说，与 FTP 都很像，如果看过 FTP 的实现，这个大概也不难理解。

当然最开始依然是用与服务器建立 socket 通信：

{% highlight python %}
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect((self.url, self.port))
{% endhighlight %}

之后会收到服务器的应答，以 163 的非 SSL 端口为例，会回复：

{% highlight bash %}
+OK Welcome to coremail Mail Pop3 Server
{% endhighlight %}

值得一提的是，POP3 中，对于命令的正确应答一般都是以`+OK`开头的，这省了不少事。

这之后列举 POP3 的常用命令：

#### USER 和 PASS

字面意思，用于认证，分别是用户名和密码，发送格式为`'USER ' + 用户名 + '\r\n' 和 `'PASS ' + 密码 + '\r\n'`，如果正确的话会返回`+OK`和邮箱状态提示。

#### STAT

显示邮箱状态，发送格式为`'STAT\r\n'`，正常的话返回`+OK`以及邮件数目和总占用空间大小。

#### LIST

返回每一封邮件的状态，发送格式为`'LIST\r\n'`，正常的话返回`+OK`以及邮件数目和总占用空间大小，之后每一行返回一个邮件的序号和他的大小，**换行符为`\r\n`**。

#### RETR

返回某一编号的邮件的全部数据，发送格式为`'RETR ' + 编号 + '\r\n'`，正常的话会直接返回邮件的所有数据，这些数据可以直接用 email 模块里的方法处理。

#### DELE

删除某一编号的邮件，发送格式为`'DELE ' + 编号 + '\r\n'`。

#### RSET

撤消删除的决定，发送格式为`'RSET\r\n'`。

#### QUIT

断开连接，并更新邮件列表（即执行刚才的删除），发送格式为`'QUIT\r\n'`。

#### NOOP

这个是向服务器表示存活的，即防超时断开，发送格式为`'NOOP\r\n'`。

### SMTP 的实现

依然是先建立 socket 连接，然后就是发送`HELO`请求或者`EHLO`请求，之后再进行其他的操作。

#### HELO 和 EHLO

HELO 或者 EHLO 在与 SMTP 服务器建立连接之后是必须，是和服务器确认身份的命令，发送格式为`'HELO ' + 邮件域 + '\r\n'` 或者 `'EHLO ' + 邮件域 + '\r\n'`，例如，和 163 邮件服务器通信，须发送`EHLO 163.com\r\n`，HELO 和 EHLO 作用虽然相同，但返回是不同的，EHLO相对于 HELO 会返回 SMTP 服务器的认证方式信息，例如：

{% highlight bash %}
250-hz-b-163smtp1.163.com
250-mail
250-PIPELINING
250-8BITMIME
250-AUTH LOGIN PLAIN
250-AUTH=LOGIN PLAIN
{% endhighlight %}

`250-AUTH LOGIN PLAIN`指明了认证方式是 LOGIN 和 PLAIN，除此之外，认证方式还有 CRAM-MD5 、 GSSAPI 、 DIGEST-MD5、 MD5，这里我们介绍前三种：

##### LOGIN 认证

LOGIN 是基于 BASE64 编码的认证。

具体步骤为：

1. 客户端发送`AUTH LOGIN\r\n`请求认证。
2. 服务器返回`334 VXNlcm5hbWU6`，后面即 BASE64 编码的`Username:`。
3. 发送 BASE64 编码的用户名。
4. 服务器返回`334 UGFzc3dvcmQ6`，后面即 BASE64 编码的`Password:`。
5. 发送 BASE64 编码的密码。
6. 服务器返回 235 ，认证成功。

##### PLAIN 认证

还是基于 BASE64 的认证。（其实感觉 BASE64 基本是防君子不防小人的东西，基本和明文区别不太大了。）

具体步骤为直接发送'AUTH PLAIN '，然后再发送BASE64编码下的`用户名+\0+用户名+\0+密码`，也有的服务器是两者直接合并发送。

##### CRAM-MD5 认证

还是基于 BASE64 的认证。

具体步骤为：

1. 客户端发送`AUTH CRAM-MD5`
2. 服务器返回 334 以及一段 BASE64 编码的值，解开后是`<24609.1047914046@popmail.Space.Net>`这种形式，我们把前者称为一个随机值。
3. 我们通过公式`MD5((密码 XOR opad), MD5((密码 XOR ipad), 随机值))`（ipad 和 opad 为 Keyed-MD5 算法规定的常数）组合一个新密文，然后把 BASE64 编码的`用户名 + ' ' + 新密文`发送过去。
4. 返回认证成功。

这基本是一种挑战式的认证，非 SSL 下感觉除了中间人攻击就没辙了。

剩下的几种可以参照[这里](http://www.fehcom.de/qmail/smtpauth.html)，如果后面有时间，我会再翻译一下= x =。

#### MAIL

MAIL 命令的格式为`'mail from: <' + 发件人 + '>'`，是设定发件人初始化邮件的命令，正常返回 250。

#### RCPT

RCPT 命令的格式为`'rcpt to: <' + 收件人 + '>'`，正常返回 250 ，如果有多个收件人，就多次发送这个命令就好。

#### DATA

DATA 命令的格式为`'DATA\r\n'`，之后接到 354 状态码就可以发送邮件内容了，如果想省事的话，这里完全可以用 email 模块里的方法生成`str`类型的邮件。最后以`\r\n.\r\n`结束。正常返回 250。（这里没有试验一次多封邮件的状况是不是多次使用 DATA 命令。）

#### RSET

放弃该邮件，格式为`'RSET\r\n'`。

#### QUIT

发送并推出，格式为`'QUIT\r\n'`。

### 完全不想提的 GUI 部分

好像没啥好说的，因为之前就写过邮件管理的 GUI ，所以这次基本也是轻车熟路了。要说注意的就还是线程安全问题。

### 最后

`SMTP` 和 `POP3` 在 Python 中也有现成的模块，可以去看看来提升一下自己代码的美观。

然后我默默地继续去复习软件安全 (:з」∠)_。

拓展阅读：  
[SMTP AUTH](http://www.fehcom.de/qmail/smtpauth.html)  
[MIME wiki 页面（众标准也在里面）](http://zh.wikipedia.org/zh-cn/%E5%A4%9A%E7%94%A8%E9%80%94%E4%BA%92%E8%81%AF%E7%B6%B2%E9%83%B5%E4%BB%B6%E6%93%B4%E5%B1%95)  
[POP3 wiki 页面（众标准也在里面）](http://zh.wikipedia.org/zh-cn/%E9%83%B5%E5%B1%80%E5%8D%94%E5%AE%9A)  
[SMTP wiki 页面（众标准也在里面）](http://zh.wikipedia.org/wiki/%E7%AE%80%E5%8D%95%E9%82%AE%E4%BB%B6%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE)  

