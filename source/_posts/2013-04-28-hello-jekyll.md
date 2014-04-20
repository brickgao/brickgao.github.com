layout: post
title: 从Wordpress迁移到jekyll
date: 2013/04/28 00:00:01
tags: 
- Jekyll
- Wordpress
categories:
- Develop
---

![jekyll](\media\2013\04\jekyll.jpg)

到今天为止终于把所有的文章全部迁移到了 Jekyll。

Jekyll 搭建的教程很多，而且也很简单，搭建步骤在此就不再多说了。

之前用过一阵 Jekyll, 大概是去年暑假到年底，那时候用 Jekyll 是因为自己的虚拟空间到期而且 wp 主页被黑，后来也是因为 Jekyll 的一些 bug，换回了 wordpress。

今年年初开始用 vim，大概花了很久才习惯了 vim 的一些键位，而且自己玩了一阵 wordpress 后，觉得 wordpress 虽然不错，但是还是太麻烦，与我喜欢的精简风格相悖。

之前用 Jekyll 写博客的时候，很懂 markdown 写的文章没有搬运，这次直接方便的移动了过来，剩下的只有处理之前的 bug 和搬运 Wordpress 上的文章。

<!-- more -->

因为自己的 wp 博客文章不多，所以采用的是手动搬运，如果想用脚本搬运，可以 google 一下，Wordpress 文章的搬运脚本很多，大多功能是基本就是把文章的`tag`, `title`, `layout`添加后, 再把wordpress中`code`标签转换为`highlight`标签。

如果你是在 win 上使用 Jekyll，你会发现 bug 很多，之前遇到的不能编译 markdown 文件的原因之一是文字编码问题，可以在`git bash`上改变环境变量来解决

``` bash
export LC_ALL=en_US.UTF-8
export LANG=en_US.UTF-8
```

剩下不能编译的问题出现在高亮插件`pygments`上，如果你的高亮代码中存在`a[] = \{\{}, {}, {}, {}\}\`(去掉\\)这样的语句，都不会被自动转义，目测都会出现无法编译的错误。

`pygments`中的`linenos`在我的page中也存在问题，这个问题可以通过插件来解决，具体的插件请在我的`repo`中查看

<del>之后在rss订阅中还存在一个bug，如果你的标题中存在`&`符号，会导致`atom.xml`的错误，目前还不知道这个错误是不是普遍发生在jekyll上，待观察。</del>通过增加`xml_escape`修复。

这次更新更换了一个主题，风格很干净，自己加了一个`fancybox`插件。

vim + markdown + jeykll + github很灵活，数据方面，数据都在 github 上托管，随时可以查看历史数据并回滚，文章方面，你可以选择离线写完后，下次push到服务器上，像现在这篇文章就是我在火车上写下的。

之后可能还有换到`hexo`的计划，折腾是蛮好的，之前在别人的 blog 上看到一句话：

    "任何的改变，都会痛，但是不能因此不去改变“

写在这里共勉。
