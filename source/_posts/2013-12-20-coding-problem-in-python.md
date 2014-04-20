layout: post
title: Python 中的一些编码问题
date: 2013-12-20 00:32:41
tags: 
- Python
- 编码
categories:
- Develop
---

最近断断续续和几个机油在写一个邮件查看器一类的东西，中间遇到了很多编码问题，想起来 Python 里的编码问题一直就没大搞清楚，趁这次出问题刚好研究了一下。

<!-- more -->

首先，我们要简单知道一下编码，编码相当于对字符的映射。简单来说，编码有 ASCII 码、 Unicode、 GB等， ASCII 码本身一共128个，无法表示所有的文字，所以 Unicode 这时就出现了，Unicode 通过拓展编码长度的方式拓展了其字符集，UTF-8是其中的一个实现，而 GB 标准是对汉字的一个专用编码，我们通常说道的 GBK 是它的一个拓展。我们一般见到的汉字是 GBK 或者 UTF-8 编码的，UTF-8 作为国际编码通用性更高，在多数时候不大会出问题，但缺点就是相对于拓展了国家标准的 GBK，占用的空间较大。

好了，简单了解了编码，我们转到 Python 里来，我这里是 Python27。Python 支持非 ASCII 字符集，不过我们要实现去声明，通常我们见到的`# -*- coding: utf-8 -*-`就是对选用字符集的声明。

通常我们输入一个字符串，可以有个两种类型`<type 'str'>`和`<type 'unicode'>`:

``` python
>>> s = '中文'
>>> s
'\xd6\xd0\xce\xc4'
>>> type(s)
<type 'str'>
>>> len(s)
4
>>> s = u'中文'
>>> type(s)
<type 'unicode'>
>>> s
u'\u4e2d\u6587'
>>> len(s)
2
```

其中`<type 'str'>`是将字符串看作字节的序列，而`<type 'unicode'>`是将字符串看作字符的序列。对于一般情况下，把中文字符串保存为 unicode 类型的最好的，这样便于我们进行字符串切片的操作，也方便我们对不同字符集的转换。

`<type 'str'>`可以通过合适的`decode()`或者`unicode()`来变成`<type 'unicode'>`，而`<type 'unicode'>`可以通过合适的`encode()`来变成`<type 'str'>`，我们可以对 unicode 字符串进行各种转码操作，可以说，unicode 是 python 字符串编码中的一个重要的中间类型。

``` python
>>> s = '中文'
>>> s
'\xd6\xd0\xce\xc4'
>>> s.decode('gbk')
u'\u4e2d\u6587'
>>> unicode(s, 'gbk')
u'\u4e2d\u6587'
>>> s = u'中文'
>>> s.encode('gbk')
'\xd6\xd0\xce\xc4'
```

而在跨平台的过程之中，我们通过`locale.setlocale(locale.LC_ALL, '')`可以获取当前系统的编码值，然后再根据系统的编码，对字符串进行编码。

最后在提一下 PyQt 中的 QString 和 QByteArray，这些是因为 PyQt 作为 Qt 的承接，为了与 C++ 中字符串一致创造的，其中 QString 与 unicode 相当，而 QByteArray 与 str 相当。


拓展阅读：  
[字符编码笔记：ASCII，Unicode和UTF-8](http://www.ruanyifeng.com/blog/2007/10/ascii_unicode_and_utf-8.html) 阮一峰的网络日志
[Unicode in Python 2.x Docs](http://docs.python.org/2/library/functions.html#unicode) Python Documentation
[Sequence Types — str, unicode, list, tuple, bytearray, buffer, xrange](http://docs.python.org/2/library/stdtypes.html#typesseq) Python Documentation  
[Where is my Character?](http://www.unicode.org/standard/where/) unicode.org  
[GB 汉字内码扩展规范](http://zh.wikipedia.org/wiki/%E6%B1%89%E5%AD%97%E5%86%85%E7%A0%81%E6%89%A9%E5%B1%95%E8%A7%84%E8%8C%83) wiki
