---
layout:     post
title:      React Fiber 分片
description: React Fiber 分片
date:       2018-11-23
author:     Eric
header-img: img/post-bg-2015.jpg
catalog: true
tags:
    - react
    - 知识点
    - 前端
---

#### 原版本
现有的React版本，当组件树很大的时候就会出现这种问题，因为更新过程是同步地一层组件套一层组件，逐渐深入的过程，在更新完所有组件之前不停止，函数的调用栈就像下图这样，调用得很深，而且很长时间不会返回。
![](https://ws1.sinaimg.cn/large/006tNbRwgy1fun4igoh6ij30k006iq30.jpg)

因为JavaScript单线程的特点，每个同步任务不能耗时太长，不然就会让程序不会对其他输入作出相应，React的更新过程就是犯了这个禁忌，而React Fiber就是要改变现状。

#### React Fiber的方式
破解JavaScript中同步操作时间过长的方法其实很简单——分片。

把一个耗时长的任务分成很多小片，每一个小片的运行时间很短，虽然总时间依然很长，但是在每个小片执行完之后，都给其他任务一个执行的机会，这样唯一的线程就不会被独占，其他任务依然有运行的机会。

React Fiber把更新过程碎片化，执行过程如下面的图所示，每执行完一段更新过程，就把控制权交还给React负责任务协调的模块，看看有没有其他紧急任务要做，如果没有就继续去更新，如果有紧急任务，那就去做紧急任务。

维护每一个分片的数据结构，就是Fiber。

有了分片之后，更新过程的调用栈如下图所示，中间每一个波谷代表深入某个分片的执行过程，每个波峰就是一个分片执行结束交还控制权的时机。

![](https://ws4.sinaimg.cn/large/006tNbRwgy1fun4igtjiij30k008ujrp.jpg)

