---
layout:     post
title:      《深入浅出Nodejs》阅读笔记
description: 《深入浅出Nodejs》阅读笔记
date:       2017-12-04
author:     Eric
header-img: img/post-bg-2015.jpg
catalog: true
tags:
    - nodejs
---

# 《深入浅出Nodejs》阅读笔记
在commonjs中require自定义模块,查找该类模块是最费时的。模块路径的生成规则如下：
-   当前文件目录下的node_modules的目录。
-   父目录下的node_modules的目录
-   父目录的父目录下的node_modules的目录
-   沿路径向上逐级递归，直到根目录下的node_modules目录
它的生成方式与JavaScript的原型链或作用域的查找方式十分类似。在加载的过程中，Node会逐个尝试模块路径中的路径，直到找到目标文件为止。可以看出，**当前文件的路径越深，模块查找耗时会越多**，这是自定义模块的加载速度是最慢的原因。

---

commonjs模块规范允许在标识符中不包含文件扩展名，这种情况下，Node会按.js、.node、.json的次序补足扩展名，依次尝试。在尝试过程中，需要调用fs模块**同步阻塞式**地判断文件是否存在。因为Node是单线程的，所以这里是一个会引起性能问题的地方。小诀窍：在传递给require的标识符中带上扩展名，会加快一点速度，同时同步配合缓存。

---

Node是单线程的，这里的单线程仅仅只是JavaScript执行在单线程中罢了。在Node中，无论是*nix还是Windows平台，内部完成I/O任务的另有线程池。

---

立即异步执行一个任务使用setTimeout(fn,0)的方式较为浪费性能，而使用process.nextTick()方法相对较为轻量。每次调用process.nextTick()方法，只会将回调函数放入队列中，在下一轮Tick时取出执行。**定时器中采用红黑树的操作时间复杂度未O(lg(n))，nextTick()的时间复杂度为O(1)**。相较之下，process.nextTick()更高效。

---

Node通过**事件驱动**的方式处理请求，无须为每一个请求创建额外的对应线程，可以省掉创建线程和销毁线程的开销，同时操作系统在调度任务时因为线程较少，上下文切换的代价很低。这使得服务器能够有条不絮地处理请求，即使在大量连接的情况下，也不受线程上下文切换开销的影响，这是Node高性能的一个原因。

---

如果对一个事件添加了超过10个监听器，将会得到一条警告。这一处设计与Node自身单线程运行有关，设计者认为监听器太多可能会导致内存泄漏，所以存在这样一条警告。调用emitter.setMaxListeners(0)；可以将这个限制去掉。另一个方面，由于事件发布会引起一系列监听器执行，如果事件相关的监听器过多，可能存在过多占用CPU的情景。

---

我们的目标是既要享受异步I/O带来的性能提升，也要保持良好的编码风格。这里以渲染页面所需要的模板读取、数据读取和本地化资源读取为例简要介绍一下，相关代码如下：

```
var count = 0;
var results = {};
var done = function(key,value){
    results[key] = value;
    count++;
    if(count === 3){
        //渲染页面
        render(results);
    }
};
fs.readFile(template_path,'utf8',function(err,template){
    done('template',template);
});
db.query(sql,function(err,data){
    done('data',data);
});
li0n.get(function(err,resources){
    done('resources',resources);
});
```
由于多个异步场景中回调函数的执行并不能保证顺序，且回调函数之间互相没有任何交集，所以需要借助一个第三方函数和第三方变量来处理异步协作的结果。通常，我们把这个用于检测次数的变量叫做哨兵变量。聪明的你也许已经想到利用偏函数来处理哨兵变量和第三方函数的关系了，相关代码如下：

```
var after = function(times,callback){
    var count = 0,result = {};
    return function(key,value){
        results[key] = value;
        count++;
        if(count === times){
            callback(results);
        }
    };
};
var done = after(times,render);
```
上述方案实现了多对一的目的。如果业务继续增长，我们依然可以继续利用发布/订阅方式来完成多对多的方案，相关代码如下：

```
var emitter = new events.Emitter();
var done = after(times,render);

emitter.on('done',done);
emitter.on('done',other);

fs.readFile(template_path,'utf8',function(err,template){
    emitter.emit('done','template',template);
});
db.query(sql,function(err,data){
    emitter.emit('done','data',data);
});
l10n.get(function(err,resources){
    emitter.emit('done','resources',resources);
});
```

这种方案结合了前者用简单的偏函数完成多对一的收敛和事件订阅/发布模式中一对多的发散。

在上面的方法中，有一个令调用者不那么舒服的问题，那就是调用者要去准备这个done()函数，以及在回调函数中需要从结果中把数据一个一个提取出来，再进行处理。

另一个方案则是EventProxy模块，它是对事件订阅/发布模式的扩充，可以自由订阅组合事件。由于依旧采用的是事件订阅/发布模式，与Node十分契合，相关代码如下：

```
var proxy = new EventProxy();
proxy.all('template','data','resources',function(template,data,resources){
    //TODO
});
fs.readFile(template_path,'utf8',function(err,template){
    proxy.emit('template',template);
});
db.query(sql,function(err,data){
    proxy.emit('data',data);
});
l10n.get(function(err,resources){
    proxy.emit('resources',resources);
});
```
EventProxy提供了一个all()方法来订阅多个事件，当每个事件都被触发之后，侦听器才会执行。另外的一个方法是tail()方法，它与all()方法的区别在于all()方法的侦听器在满足条件之后只会执行一次，tail()方法的侦听器则在满足条件时执行一次之后，如果组合事件中的某个事件被再次触发，侦听器会用最新的数据继续执行。

all()方法带来的另一个改进则是：在侦听器返回数据的参数列表与订阅组合事件的事件列表是一致对应的。

除此之外，在异步的场景下，我们常常需要从一个接口多次读取数据，此时触发的事件名或许是相同的。EventProxy提供了after()方法来实现事件在执行多少次后执行侦听器的单一事件组合订阅方式，示例代码如下：

```
var proxy = new EventProxy();
proxy.after('data',10,function(datas){
    //TODO
});
```
这段代码表示执行10次data事件后执行侦听器。这个侦听器得到的数据为10次按事件触发次序排序的数组。

EventProxy模块除了可以应用于Node中外，还可以用在前端浏览器中。

---

通过预先转换静态内容为Buffer对象，可以有效地减少CPU的重复使用，节省服务器资源。在Node构建的Web应用中，可以选择将页面中的动态内容和静态内容分离，静态内容部分可以通过预先转换为Buffer的方式，使性能得到提升。由于文件自身是二进制数据，所以在不需要改变内容的场景下，尽量只读取Buffer，然后直接传输，不做额外的转换，避免损耗。

---

提到Node，不能错过的是WebSocket协议。它与Node之间的配合堪称完美，其理由有两条。
-  WebSocket客户端基于事件的编程模型与Node中自定义事件相差无几。
-  WebSocket实现了客户端与服务端之间的长连接，而Node事件驱动的方式十分擅长与大量的客户端保持高并发连接。
除此之外，WebSocket与传统HTTP有如下好处。
-  客户端与服务器端只建立一个TCP连接，可以使用更少的连接。
-  WebSocket服务器端可以推送数据到客户端，这远比HTTP请求响应模式更灵活、更高效。
-  有更轻量级的协议头，减少数据传送量。

---


**为静态组件使用不同的域名**

为不需要Cookie的组件换个域名可以实现减少无效Cookie的传输。所以很多网站的静态文件会有特别的域名，使得业务相关的Cookie不再影响静态资源。当然换用额外的域名带来的好处不只这点，还可以突破浏览器下载线程数量的限制，因为域名不同，可以将下载线程数翻倍，但是使用额外域名还是有一定的缺点的，那就是将域名转换为IP需要进行DNS查询，多一个域名就多一次DNS查询。

**减少DNS查询**

看起来减少DNS查询和使用不同的域名是冲突的两条规则，但是好在现今的浏览器都会进行DNS缓存，以削弱这个副作用的影响。
Cookie除了可以通过后端添加协议头的字段设置外，在前端浏览器中也可以通过JavaScript进行修改，浏览器将Cookie通过document.cookie暴露给了JavaScript。前端在修改Cookie之后，后续的网络请求中就会携带上修改过后的值。

---

为了提高性能，有几条关于缓存的规则。
-  添加Expires或Cache-Control到报文头中。
-  配置ETags。
-  让Ajax可缓存。

**在请求报文中附带If-Modified-Since字段，它将询问服务器端是否有更新的版本，本地文件的最后修改时间。如果服务器端没有新的版本，只需想赢一个304状态码，客户端就使用本地版本。**


---

ETag的全称是Entity Tag，由服务器端生成，服务器端可以决定它的生成规则。如果根据文件内容生成散列值，那么条件请求讲不会受到时间戳改动造成的带宽浪费。

与If-Modified-Since/Last-Modified不同的是，ETag的请求和响应是If-None-Match/ETag，浏览器在收到ETag：“83-1359871272000”这样的请求后，在下次的请求中，会将其放置在请求头中：If-None-Match：“83-1359871272000”。

Expires是一个GMT格式的时间字符串。浏览器在接到这个过期值后，只要本地还存在这个缓存文件，在到期时间之前它都不会再发起请求。**但是Expires的缺陷在于浏览器与服务器之间的时间可能不一致，这可能会带来一些问题，比如文件提前过期，或者到期后并没有被删除。**

Cache-Control比Expires优秀的地方在于能够避免浏览器与服务器端时间不同步带来的不一致性问题，只要进行类似倒计时的方式计算过期时间即可。除此之外，Cache-Control的值还能设置public、private、no-cache、no-store等能够更精细地控制缓存的选项。在浏览器中如果两个值同时存在，且被同时支持时，Cache-Control设置的max-age值会覆盖Expires。