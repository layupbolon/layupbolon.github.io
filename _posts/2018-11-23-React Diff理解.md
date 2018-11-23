---
layout:     post
title:      理解React Diff
description: 理解React Diff
date:       2018-11-23
author:     Eric
header-img: img/post-bg-2015.jpg
catalog: true
tags:
    - react
    - 知识点
    - 前端
---

### 传统diff算法
> 传统 diff 算法通过循环递归对节点进行依次对比，效率低下，算法复杂度达到 O(n^3)，其中 n 是树中节点的总数。O(n^3) 到底有多可怕，这意味着如果要展示1000个节点，就要依次执行上十亿次的比较。代价太高。

### React diff优化
> 传统的diff算法的复杂度为O(n^3),显然无法满足性能要求。Facebook工程师通过大胆的策略，将O(n^3)复杂度简化成了O(n),怎么做到的呢？

### diff策略
- Web UI 中 DOM 节点跨层级的移动操作特别少，可以忽略不计。
- 拥有相同类的两个组件将会生成相似的树形结构，拥有不同类的两个组件将会生成不同的树形结构。
- 对于同一层级的一组子节点，它们可以通过唯一 id 进行区分。
基于以上三个前提策略，React团队对传统diff算法优化基于三个策略（《深入React技术栈》等讲的确实有点难理解且模糊，这边经过理解给出了自己的理解）
- a->tree diff
- b->component diff
- c->element diffd

#### 优化策略a: tree diff
基于tree diff策略，React对Virtual DOM树进行 分层比较、层级控制，只对相同颜色框内的节点进行比较(同一父节点的全部子节点)，当发现某一子节点不在了直接删除该节点以及其所有子节点，不会用于进一步的比较，在算法层面上就是说只需要遍历一次就可以了，而无需在进行不必要的比较，便能完成整个DOM树的比较。
如图：
![image](https://ws4.sinaimg.cn/large/006tNc79gy1ft82snay5rj30uw0fytjd.jpg)

同属于***分层比较、层级控制***范畴，还会出现DOM节点跨层级的移动操作(React中这种情况DOM节点不稳定，损害性能，所以开发中不推荐这种情况的出现)，React diff怎么解决的呢？如下图情况：
![image](https://ws1.sinaimg.cn/large/006tNc79gy1ft82sn17i3j30xw0gwdk1.jpg)

上面描述的是同一层次不同DOM节点范畴，React diff用趋近于‘暴力’的方式，并不是把A B C 直接拼接到 D 节点上，而是删除A B C 三个节点之后在 D 下面在创建的 A B C。这里不做详细分析，想直观理解该过程，建议阅读[这篇用在生命周期里打log的方式展示上述过程](https://www.jianshu.com/p/fa4ca1fed4cf)

#### 优化策略b: component diff
React是基于组件构建应用的，对于组件间的比较所采用的策略也是简洁高效。

- 对于同一类型的组件，根据Virtual DOM是否变化也分两种，可以用shouldComponentUpdate()判断Virtual DOM是否发生了变化，若没有变化就不需要在进行diff，这样可以节省大量时间，若变化了，就对相关节点进行update
- 对于非同一类的组件，则将该组件判断为 dirty component，从而替换整个组件下的所有子节点。

如下图，当 component D 改变为 component G 时，即使这两个 component 结构相似，一旦 React 判断 D 和 G 是不同类型的组件，就不会比较二者的结构，而是直接删除 component D，重新创建 component G 以及其子节点。虽然当两个 component 是不同类型但结构相似时，React diff 会影响性能，但正如 React 官方博客所言：不同类型的 component 是很少存在相似 DOM tree 的机会，因此这种极端因素很难在实现开发过程中造成重大影响的。
而如果上图中左一中的D节点只是单纯的改变什么state，update就好了。

![image](https://ws3.sinaimg.cn/large/006tNc79gy1ft82smsgbij30v20ckq8w.jpg)

#### 优化策略c: element diff
所有同一层级的子节点.他们都可以通过key来区分-----并遵循策略a、b。
![image](https://ws4.sinaimg.cn/large/006tNc79gy1ft82sm1ptbj30s80h8dlb.jpg)

没经过优化的算法，实现新老交替的方法是将A B C D全部删除之后，在新建B A D C，这样的实现方法显然很垃圾，React diff怎么优化呢？是通过为每一个节点添加key值标识。

新老集合所包含的节点，如上图所示，新老集合进行 diff 差异化对比，通过 key 发现新老集合中的节点都是相同的节点，因此无需进行节点删除和创建，只需要将老集合中节点的位置进行移动，更新为新集合中节点的位置，此时 React 给出的 diff 结果为：B、D 不做任何操作，A、C 进行移动操作，即可。
上述分析的是新老集合中存在相同节点但是位置不同，要是有新加入的节点且有旧节点需要删除呢？这里不再啰嗦，如下图：
![image](https://ws4.sinaimg.cn/large/006tNc79gy1ft82slqu62j30q00hijx2.jpg)

当然，React diff 还是存在些许不足与待优化的地方，如下图所示，若新集合的节点更新为：D、A、B、C，与老集合对比只有 D 节点移动，而 A、B、C 仍然保持原有的顺序，理论上 diff 应该只需对 D 执行移动操作，然而由于 D 在老集合的位置是最大的，导致其他节点的 _mountIndex < lastIndex，造成 D 没有执行移动操作，而是 A、B、C 全部移动到 D 节点后面的现象。

![image](https://ws3.sinaimg.cn/large/006tNc79gy1ft82slgrf0j30s40gk0zb.jpg)

###### 加了key的好处：
如果不加key，map遍历的时候控制台发出warn，既然是warn就说明不加也能实现遍历，但是是经过删除、创建、插入实现，这样的话损害性能可想而知，而加上key就可以有助于React diff算法结合Virtual DOM找到最合适的方式进行diff，最大限度的实现高效diff，即那里需要改变，就改变哪里！

### 总结
- React 通过分层求异的策略，对 tree diff 进行算法优化；
- React 通过相同类生成相似树形结构，不同类生成不同树形结构的策略，对 component diff 进行算法优化；
- React 通过设置唯一 key的策略，对 element diff 进行算法优化；
- React 通过制定大胆的 diff 策略，将 O(n3) 复杂度的问题转换成 O(n) 复杂度的问题；
- 建议，开发时保持稳定的DOM结构有助于性能的提升；
- 建议，在开发过程中，尽量减少类似将最后一个节点移动到列表首部的操作，当节点数量过大或更新操作过于频繁时，在一定程度上会影响 React 的渲染性能。