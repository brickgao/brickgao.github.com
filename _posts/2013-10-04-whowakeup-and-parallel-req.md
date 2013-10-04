---
layout: post
title: WhoWakeUp的并发处理
disqus: y
date: 2013-10-04 17:30:37
tags: [Node.js, MongoDB, Express]
---

![禁忌「フォーオブアカインド」](/media/2013/09/Flandre_Scarlet.jpg)

WhoWakeUp是我帮机油写的一个起床签到的小应用，用于微信平台。

这个小应用是用来响应`sitename/req?id=`的请求，然后返回签到的信息。微信平台要做的只是发去Get请求，然后爬网页即可。

最早参照doc别人写的应用写出了一个版本，只是简单测试之后就挂到BAE上去了，因为BAE有本身的优化，就没做太多的处理。

后来被提醒起床签到这种应用是一个时间密集而且需要rank唯一的应用，如果相应并发的请求出现问题就没有意义了。

后来就用Python写了这么一个并发访问的[脚本(戳我)](https://gist.github.com/brickgao/6812931)来测试，运行后立即就报500的错误并提示

	500 Error: db object already connecting,
    open cannot be called multiple times

看了报错信息和代码并且参考了[这篇文章](http://cnodejs.org/topic/5190d61263e9f8a542acd83b)发现还是数据库的连接与断开出了问题。

在很多例子中数据库的访问是这么写的：

{% highlight js %}
var mongodb = require("mongodb"),
    mongoserver = new mongodb.Server('localhost', 27017,server_options ),
    db = new mongodb.Db('test', mongoserver, db_options);

db.open(function(err, db) {
  if(err)
    return callback(err);
  db.colleciton('whowakeup').save({test:1}, function(err, result) {
    db.close();
    callback(null);
  });
});
{% endhighlight %}

程序是通过连接池与db联系的，在默认的情况下，数据库中存在poolSize选项，默认值是5，也就是说默认有5个连接池。

本地跑程序是不会出问题的，但一旦到了http server中，只有5个连接池会对性能造成很大的影响，更关键的是`mongodb.open`和`mongodb.close`这两个方法会在高并发访问时使程序出现上文所提到的'数据库对象不能被多次打开'的隐性错误。

我们可以这么改，在程序启动的时候用`mongodb.open()`对数据库对象打开一次，每次直接访问数据库，之后扔掉`mongodb.open/close`这两个方法。

但是这样又会有问题，当并发访问的个数大于数据库的连接限制时，就会存在阻塞。

所以说直接引用db对象并不好，在这里我们可以重用`genericpool`这个模块来动态限制连接数。

{% highlight js %}
var settings = require('../settings'),
    Db = require('mongodb').Db,
    Connection = require('mongodb').Connection,
    Server = require('mongodb').Server,
    poolModule = require('generic-pool');

var pool = poolModule.Pool({
    name     : 'mongodb',
    create   : function(callback) {
        var server_options={'auto_reconnect':false, poolSize:1};
        var db_options={w:-1};
        var mongoserver = new Server(settings.host, Connection.DEFAULT_PORT, server_options );
        var db = new Db(settings.db, mongoserver, db_options);
        db.open(function(err,db) {
            if(err) return callback(err);
            callback(null, db);
        });
    },
    destroy  : function(db) { db.close();},
    max      : 10,
    idleTimeoutMillis : 30000,
    log : false 
});
{% endhighlight %}

其中max规定了最大的连接数，在WhoWakeUp中，为了防止在并发时多次修改数据库而产生污染，牺牲了性能，将连接数置为1。

之后每次调用`pool.acquire`来连接，用`pool.release`来释放即可。

后来翻了一下doc，发现官方的doc中已经是`mongodb.connect`来连接了，而且又发现了Mongoose这个Node.js对MongoDB的驱动，更加符合我的习惯一些，好像不用人为处理open和close，之后可能会用用看。


