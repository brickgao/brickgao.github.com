layout: post
title: 模拟登陆,提交表单与网页抓取
date: 2013/05/13 00:00:01
tags: 
- Python
- Request
- Http
- 模拟登录
- 提交表单
- 网页抓取
categories:
- Diary
---

这两天忽然想起来学着写一下网页请求相关的东西，写了几天来总结一下。

<!-- more -->

##网页请求

用Python来写对网页请求的话有两种选择

*   使用`urllib2`, `urllib`模块来写  
*   使用`Request`模块来写

个人比较推荐用第二种方法来写，原因后面再说。

###利用`urllib2`和`urllib`来写

这种方法先要处理`Cookies`，例如：

``` python
cookie = cookielib.CookieJar()
cookies = urllib2.HTTPCookieProcessor(cookie)
opener = urllib2.build_opener(cookies)
urllib2.install_opener(opener)
```

然后再对网页发出请求，这里以模拟登录从网位例：

``` python
parms = {
        'email': mail,
        'password': password,
        'domain': 'renren.com',
}
            
loginURL = 'http://renren.com/PLogin.do'
login = urllib2.urlopen(loginURL, urllib.urlencode(parms))
```

这里的`parm`是发送的内容，具体每个网页发送的内容你可以用chrome的开发者工具中的Network查看，向哪个网页请求以达到操作也可以在Network中看到。

###利用`Request`来写

如果你没有安装Request模块，可以通过`esay_install Request`来安装，如果没有`esay_install`，请谷歌一下安装方法。

`Request`模块相对于`urllib2`, `urllib`更简单一些，它把大部分的工作做了，不需要用户去考虑太多的问题，这也是我推荐用它的原因。

对于相同的操作，即登录从网，操作是这样的：

``` python
loginurl = 'http://renren.com/PLogin.do'
payload = {
    'email': mail,
    'password': password,
    'domin': 'renren.com',
}
renren = requests.post(loginurl, params = payload)
```

更多的`Request`的功能你可以从它的[项目页](http://docs.python-requests.org/en/latest/)上找到。

##关于抓取网页及处理

网页抓取基本是和网页请求衔接着的。

###抓取网页

如果你采用的是`urllib`和`urllib2`，网页信息可以通过`urllib2.urlopen().read()`来读取。

如果你采用的是`Request`，网页信息可以通过`requests.text`来读取。

###处理信息

关于信息的筛选推荐使用`pyquery`来实现。

`pyquery`的操作基本和`jQuery`一致，如果你学过`jQuery`，那简单翻阅一下文档就行了。

比如我这里要筛选`class = content-main`的部分

``` python
d = PyQuery(file)
print(d('.content-main').text())
```

pyquery的文档请戳[这里](http://pythonhosted.org/pyquery/)

近期如果开了一个命令行来控制从网的坑，如果有时间的话，应该会最近填上的。
