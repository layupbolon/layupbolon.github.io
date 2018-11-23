---
layout:     post
title:      Immutable.js与React,Redux及reselect的实践
description: Immutable.js与React,Redux及reselect的实践
date:       2018-11-23
author:     Eric
header-img: img/post-bg-2015.jpg
catalog: true
tags:
    - react
    - redux
    - immutable
    - 前端
---

### Immutable.js解决的问题
React通过对组件属性（props）和状态（state）进行变更检查以决定是否更新并重新渲染该组件，若组件状态太过庞大，组件性能就会下降，因为对象越复杂，其相等性检查就会越慢。
1. 对于嵌套对象，必须迭代层层进行检查判断，耗费时间过长；
2. 若仅修改对象的属性，其引用保持不变，相等性检查中的引用检查结果不变；

Immutable提供一直简单快捷的方式以判断对象是否变更，对于React组件更新和重新渲染性能可以有较大帮助。

### Immutable数据
Immutable对象和原生JavaScript对象的主要差异可以概括为以下两点：
1. 持久化数据结构（Persistent data structures）
2. 结构共享（Structures sharing Trie）

##### 持久化数据结构
持久数据结构主张所有操作都返回该数据结构的更新副本，并保持原有结构不变，而不是改变原来的结构。通常利用Trie构建它不可变的持久性数据结构，它的整体结构可以看作一棵树，一个树节点可以对应代表对象某一个属性，节点值即属性值。

##### 结构共享
一旦创建一个Immutable Trie型对象，我们可以把该Trie型对象想象成如下一棵树，在之后的对象变更尽可能的重用树节点：
![image](https://ws3.sinaimg.cn/large/006tNc79gy1ft830utr6nj30u80qegxy.jpg)

当我们要更新一个Immutable对象的属性值时，就是对应着需要重构该Trie树中的某一个节点，对于Trie树，我们修改某一节点只需要重构该节点及受其影响的节点，即其祖先节点，如上图中的四个绿色节点，而其他节点可以完全重用。

### 如何使用Immutable
当使用Redux作React应用状态管理容器时，我们通常将组件分为容器组件和展示型组件，Immutable与Redux组件的实践也主要围绕这两者。

1. 除了在展示型组件内，其他地方一律使用Immutable方式操作状态对象；为了保证应用性能，在容器组件，选择器（selectors），reducer函数，action创建函数，sagas和thunks函数内等所有地方均使用Immutable，但是不在展示型组件内使用。
2. 在容器组件内使用Immutable。容器组件可以使用react-redux提供的connect方法访问redux的store，所以我们需要保证选择器（selectors）总是返回Immutable对象，否则，将会导致不必要的重新渲染。另外，我们可以使用诸如reselect的第三方库缓存选择器（selectors）以提高部分情景下的性能。
3. ==绝对不要在mapStateToProps方法内使用toJS()方法。== toJS()方法每次会调用时都是返回一个原生JavaScript对象，如果在mapStateToProps方法内使用toJS()方法，则每次状态树（Immutable对象）变更时，无论该toJS()方法返回的JavaScript对象是否实际发生改变，组件都会认为该对象发生变更，从而导致不必要的重新渲染。
4. ==绝对不要在展示型组件内使用toJS()方法。== 如果传递给某组件一个Immuatble对象类型的prop，则该组件的渲染取决于该Immutable对象，这将给组件的重用，测试和重构带来更多困难。
5. 当容器组件将Immutable类型的属性（props）传入展示型组件时，需使用高阶组件（HOC）将其转换为原生JavaScript对象。
> 该高阶组件定义如下：

```
import React from 'react';
import { Iterable } from 'immutable';

export const toJS = WrappedComponent => wrappedComponentProps => {
    const KEY = 0;
    const VALUE = 1;
    const propsJS = Object.entries(wrappedComponentProps)
    .reduce((newProps, wrappedComponentProp) => {
        newProps[wrappedComponentProp[KEY]] = Iterable.isIterable(wrappedComponentProp[VALUE]) ?
        wrappedComponentProp[VALUE].toJS() : wrappedComponentProp[VALUE];
        
        return newProps;
    }, {});
    
    return <WrappedComponent {...propsJS} />
```
该高阶组件内，首先使用Object.entries方法遍历传入组件的props，然后使用toJS()方法将该组件内Immutable类型的prop转换为JavaScript对象，该高阶组件通常可以在容器组件内使用，使用方式如下：

```
import { connect } from 'react-redux';
import { toJS } from './to-js';
import DumbComponent from './dumb.component';

const mapStateToProps = state => {
    return {
        obj:getImmutableObjectFromStateTree(state)
    }
}
export default connect(mapStateToProps)(toJS(DumbComponent));
```
这类高阶组件不会造成过多的性能下降，因为高阶组件只在被连接组件（通常即展示型组件）属性变更时才会被再次调用。你也许会问既然在高阶组件内使用toJS()方法必然会造成一定的性能下降，为什么不在展示型组件内也保持使用Immutable对象呢？事实上，相对于高阶组件内使用toJS()方法的这一点性能损失而言，避免Immutable渗透入展示型组件带来的可维护性，可重用性及可测试性是我们更应该看重的。

### Immutable的常见问题
###### 很难进行内部协作
Immutable对象和JavaScript对象之间存在的巨大差异，使得两者之间的协作通常较麻烦，而这也正是许多问题的源头。

1. 使用Immutable.js后我们不再能使用点号和中括号的方式访问对象属性，而只能使用其提供的get,getIn等API方式；
2. 不再能使用ES6提供的解构和展开操作符；
3. 和第三方库协作困难，如lodash和JQuery等。

###### 渗透整个代码库
Immutable代码将渗透入整个项目，这种对于外部类库的强依赖会给项目的后期带来很大约束，之后如果想移除或者替换Immutable是很困难的。

###### 不适合经常变更的简单状态对象
Immutable和复杂的数据使用时有很大的性能提升，但是对于简单的经常变更的数据，它的表现并不好。

###### 切断对象引用将导致性能低下
Immutable最大的优势是它的浅比较可以极大提高性能，当我们多次使用toJS方法时，尽管对象实际没有变更，但是它们之间的等值检查不能通过，将导致重新渲染。更重要的是如果我们在mapStateToProps方法内使用toJS将极大破坏组件性能，如果真的需要，我们应该使用前面介绍的高阶组件方式转换。

###### 难以调试
当我们审查一个Immutable对象时，浏览器会打印出Immutable.js的整个嵌套结构，而我们实际需要的只是其中小一部分，这导致我们调试较困难，可以使用Immutable.js Object Formatter浏览器插件解决。