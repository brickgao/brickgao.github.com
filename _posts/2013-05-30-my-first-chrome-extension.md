---
layout: post
title: 自己写的第一个Chrome Extension
disqus: y
tags: [Chrome Extension, js, Twitter]
---

前几天忽然想写个Chrome Extension玩，于是就动手写了。

也没有特别好的想法，于是写了一个Twitter停留计时和提醒的Extension。

现看了看Chrome的官方文档，后来发现有人自己翻译了一下([戳我](https://crxdoc-zh.appspot.com/extensions/getstarted.html))，翻译的还不错。

Chrome Extension大概的组成元素是:

*   一个清单文件  
*   一个或多个 HTML 文件（除非扩展程序是一个主题背景）  
*   可选：一个或多个 JavaScript 文件  
*   可选：您的扩展程序需要的任何其他文件，例如图片

发现清单列表`manifest.json`已经发展到了版本二，请务必看下版本二的改动，尤其是安全方面的改动，对于js做了一些限制，比如不能直接内联js等等。

自己写的Extension大概分为两个部分，一个是计时，一个是Time out之后的提醒。

在`background.js`中的计时部分用了`chrome.runtime.onInstalled.addListener`在拓展更新的时候来初始化各种变量。

计时模块写个两个函数一个是`startTimer()`表示启动计时器，另一个`endTimer()`停止计时器。

计时部分因为对`setTimeout("interval();")`安全限制，循环改为`setTimeout(function() {interval();},1000);`。

因为只对当前的页面是不是`Twitter.com`计时，所以利用以下函数监控`Tabs`，然后决定是否计时：

*   function checkByTabid(tabId) {} //Check the tab is twitter or not  
*   chrome.tabs.onCreated.addListener(function(tab){}) //Check when create new tab  
*   chrome.tabs.onSelectionChanged.addListener(function(tabId, changeInfo){}) //Check when select tab  
*   chrome.tabs.onRemoved.addListener(function(tabId){}) //Check when change tab  
*   chrome.tabs.onUpdated.addListener(function(tabId, changeInfo){}) //Check when update tab  
*   chrome.windows.onFocusChanged.addListener(function(windowId){}) ////Check when window change

不知道为什么参数比较的时候默认成了字符串比较，后来用`parseInt`函数修改了下。

对于限制时间的设置由HTML5中的`localStorage`来储存，一开始以为是单个Page拥有独立的`localStorage`，后来才发现是整个Extension公用，被自己坑了一下。

写桌面通知要在清单中声明可以使用，再对使用的图片声明为可使用的资源，桌面通知大概格式是

{% highlight js %}
function show() {
    var notification = webkitNotifications.createNotification(
        'icon.png',  // url of icon
        'Notice',  // title of the notice
        'Time you used on twitter is over toady >_<'  // the text of notice
    );
    notification.show();
}
{% endhighlight %}

写设置不一样的是没法对`onclick`在`button`中定义，需要对`button`的动作监听：

{% highlight js %}
document.addEventListener('DOMContentLoaded', function () {
    document.querySelector('#saveset').addEventListener('click', save_options);
});
{% endhighlight %}

写完发现有个Bug，如果用`Gogent`来访问Twiiter就会获取`tab.url`错误，不知道是为什么，之后再研究下。

这次还是新看了不少东西，感觉第一次写Extension，代码风格实在是看不过去。。。

Extension被我扔到Github上了，项目地址[https://github.com/brickgao/twittermaid](https://github.com/brickgao/twittermaid)
