layout: post
title: 中间人攻击初探
date: 2013-11-17 17:19:45
tags: 
- Man-in-the-middle attack
- Network
- 中间人攻击
categories:
- Develop
---

![今天也要努力地买萌](\media\2013\11\Hinanawi_Tenshi.jpg)

这几天在做中间人攻击相关的网络安全实验。

<!-- more -->

最早知道中间人攻击这个词的时候是在今年年初，发现访问 Github 的时候显示证书无效。

简单点来说中间人攻击就是网络你和网络之间的第三人，攻击者充当了 client 和 server 之间的转发者，作为转发者他可以修改通信中的内容，而 client 和 server 不会知道中间有转发者，双方均认为是与对方直接通信，对于加密的通信攻击实例如下（我抄wiki了啊> <）：

> alice == "嗨，Bob，我是Alice。给我你的公钥" ==> Mallory                  Bob

> alice                  Mallory == "嗨，Bob，我是Alice。给我你的公钥" ==> Bob

> alice                                       Mallory <== [ Bob 的公钥] == Bob

> alice <== [ Mallory 的公钥] == Mallory                                   Bob

> alice == "我们在公共汽车站见面！" [使用 Mallory 的公钥加密] ==> Mallory  Bob

> alice                  Mallory == "在家等我！" [使用 Bob 的公钥加密] ==> Bob

这种情况下， Bob 认为信息是直接从 Alice 哪里传过来的。

具体实现我选择了 ettercap 这个工具，具体的环境是在同一台路由器下的几台电脑，选择的是在局域网内的 dns 欺诈攻击和攻击 ssl 来嗅探密码。

安装参照[文档](https://github.com/Ettercap/ettercap " ettercap 的文档")。

我们通过修改`etter.dns`来控制攻击的结果，之后用图形化界面或者终端扫描ip段之后选择攻击对象然后我们开启 arp 和 dns 欺诈攻击。

![dns 欺诈](/media/2013/11/mitm.png)

这里我将`*.com`全部定向到了`202.114.64.60`。

攻击 ssl 的功能通过修改`etter.ssl.crt`就可以了，但是当我们访问`https://*`的页面的时候以为 PKI 的原因会很容易被发现，即提示证书无效（说到这里，如果你使用过 goagent ，你应该很熟悉这个错误， GAE 不支持原生的 ssl ，只能自己做认证，在这里它充当了一个中间人）。

这里找到了一个好一些的方式，通过`sslstrip`的 arp 欺诈把 https 全部定向为 http ，再通过 ettercap 来过滤密码，但是在这种情况下稍稍细心还是很好发现的。

而对中间人攻击的防范我们可以加强 PKI 的建设，采用一次一密或者说通过时间戳的验证等都可以。
