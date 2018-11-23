---
layout:     post
title:      js模块规范commonjs、AMD、CMD的异同
description: js模块规范commonjs、AMD、CMD的异同
date:       2018-11-23
author:     Eric
header-img: img/post-bg-2015.jpg
catalog: true
tags:
    - javascript
    - 知识点
    - 前端
---

#### CommonJS

CommonJS定义的模块分为:
- {模块引用(require)} 
- {模块定义(exports)} 
- {模块标识(module)}

> CommonJS是用在服务器端的，同步的，如nodejs

#### AMD
AMD就只有一个接口：define(id?,dependencies?,factory);

RequireJS就是实现了AMD规范

#### AMD与CMD的异同

> AMD, CMD是用在浏览器端的，异步的，如requirejs和seajs。其中，AMD先提出，CMD是根据commonjs和AMD基础上提出的。

CMD和AMD的区别有以下几点：
- 对于依赖的模块AMD是加载完立即执行，CMD是延迟执行。不过RequireJS从2.0开始，也改成可以延迟执行（根据写法不同，处理方式不通过）。
- ==CMD推崇依赖就近，AMD推崇依赖前置。==