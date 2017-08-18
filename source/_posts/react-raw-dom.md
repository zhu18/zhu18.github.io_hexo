---
title: 原生DOM操作原来可以这么快
date: 2017-07-18 14:53:30
tags: [react, javascript, dom]
---

[原文地址](https://www.zhihu.com/question/31809713/answer/54833728)
网上都说操作真实dom怎么怎么慢，但是下面这个链接案例中原生的方式却是最快的
<a href="https://link.zhihu.com/?target=http%3A//chrisharrington.github.io/demos/performance/" class=" external" target="_blank" rel="nofollow noreferrer"><span class="invisible">http://</span><span class="visible">chrisharrington.github.io</span><span class="invisible">/demos/performance/</span><span class="ellipsis"></span><i class="icon-external"></i></a>
![性能测试](/uploads/react-raw-dom1.png)
我在本地也写了个例子循环2000个随机数组，点击按钮重新生成随机数组渲染页面，也是自己用的js 操作dom 比用react 和angular 都要快，这个是怎么回事。测试方式不对，页面元素太少了，还是哪里问题，请帮忙解答下，谢谢。


鉴于有人表示没有看懂。 下面是结论:

*   题主测试结果反常是因为测试用例构造有问题。
*   React.js 相对于直接操作原生DOM有很大的性能优势， 背后的技术支撑是基于virtual DOM的batching 和diff.
*   React.js 的使用的virtual DOM 在Javascript Engine中也有执行效率的优势。

<!-- more -->


看了好几个答案感觉都没有答到点子上面。如果一个框架发明出来就是为了提高性能，那就一定要在性能上面有所体现。编程模型类型框架实在是太多太多，React.js仅仅因为这个不可能会那么受好评。

React.js仅仅是做MVC中的View，也就是渲染层，任务其实非常单一的，也没有那么多哲学方面的东西需要承载。

**1.题主测试结果反常是因为测试用例构造有问题。**

我们来看看题主的测试用例(由于题主问的react.js 和原生，所以其他的我都去掉了).
_以下所有的代码都在:_
_
[<span class="invisible">http://</span><span class="visible">fredrikluo.github.io/te</span><span class="invisible">stcases/reactjstest.html</span><span class="ellipsis"></span>](https://link.zhihu.com/?target=http%3A//fredrikluo.github.io/testcases/reactjstest.html)_

原生 (只保留了关键代码，其他都去掉了，这样看得更清楚一点）


![](/uploads/raw1.png)这里看出问题没？

整个测试用例的模式是：

1.  构造一个 String, 包含一个1000个 Tag。

2.  把这1000个 Tag 一次性添加到 DOM 树中。

其实，**题主的测试用例只做了一次 DOM 操作**。。而且主要问题是，如果你真的做一个WebAPP, 然后直接操作DOM更新，更新的模式完全不是这样子的。

在现实中，更新模式更像这个样子滴：
同样是 1000 元素需要更新， 你的界面上面分成了20个逻辑区域（或者层，或者 view, 或者whatever 框架取的名字）, **每个区域 50 个元素。在界面需要更新的时候，每个逻辑区域分别操作 DOM Tree 更新**。

那么代码看起来更像是这样子的:

![](/uploads/raw2.png)
然后我们再来看看结果

![](/uploads/raw3.png)
天了撸， 发生了什么， 原生的怎么慢这么多。（React.js 并不需要修改，无论如何每个区域都是把新的操作作用在Virtual DOM上面，然后每帧只会调用一次Render)。

**2\. React.js 相对于直接操作原生DOM有很大的性能优势， 背后的技术支撑是基于virtual DOM的batching 和diff。**

原生DOM 操作慢吗？

做为一个浏览器内核开发人员， 我可以负责任的告诉你，慢。
你觉得只是改变一个字符串吗？
我们来看看你在插入一个DOM元素的时候，发生了什么.

![](/uploads/raw4.png)
实际上，浏览器在收到更新的DOM Tree需要立即调用HTML Parser对传入的字符串进行解析，这个，呃，耗的时间可不是字符串替换可以比的哦 .

(_其实你们已经处于一个好时代了，换做几年前，浏览器还可能会花几秒到几分钟给你Reflow一下）_

这个例子还算简单的了，如果你插入的标签里面包括了脚本，浏览器可能还需要即时编译的你脚本(JIT).

_这些时间都算在你的DOM操作中的哦**。**_

我们再来看看统计。

![](/uploads/raw5.png)
131 ms **都花在了Loading 里面(ParseHTML)**。

_另外注意一些细节，Profiler 报告整个函数使用了418ms, 因为有些时间在JS里面是统计不到的，比如Rendering的时间， 所以， 多用Profiler._

我们再来看一个图， 这个原测试用例的Profiling， 1000 Tag 一次插入

![](/uploads/raw6.png)
呃，如果你还没有看出端倪的话，我提示一下: 这里的解析时间(Loading)降到了13 ms.

_**同样的数据（1000 元素），分20次解析， 每次解析50个，耗时是 一次性解析的 10 倍左右。。也就是说，有9倍开销，都花在了DOM函数，以及背后的依赖的函数本身的开销上了。** DOM 操作本身可能不耗时，但是建立上下文，层层传递的检查的开销就不容小视了_**。**

这个官方称为”API Overhead”. 改变DOM 结构的调用都有极其高的API Overhead.
而对于Overhead高的API，标准解决办法就是两个：

Batching 和 Diff.

Diff 大家都比较了解， Batching是个啥？

想象在一个小山村里面，只有一条泥泞的公路通向市区，公交车班次少，每次要开半个小时，如何保证所有乘客最快到达目的地？

_搜集尽量多的乘客，然后一次性的把它们运往市区， 这个就是Batching._

如果你搜下React.js 的文档里面。这里有专门提到：

> “ You may be thinking that it's expensive to change data if there are a large number of nodes under an owner. The good news is that JavaScript is fast and `render()` methods tend to be quite simple, so in most applications this is extremely fast. Additionally, the bottleneck is almost always the DOM mutation and not JS execution. React will optimize this for you using batching and change detection.“

_这个才是React.js 引入Virtual DOM 的精髓之一， **把所有的DOM操作搜集起来，一次性提交给真实的DOM**._

Batching 或者 Diff, 说到底，都是为了尽量减少对慢速DOM的调用。

_类似技术在游戏引擎里面（对 OPENGL 的Batch), 网络传输方面( 报文的Batch), 文件系统读写上都大量有使用。_

**3\. React.js 的使用的virtual DOM在Javascript Engine中也有执行效率的优势**

Virtual DOM 和 DOM 在JS中的执行效率比较**。**
-------------------------------------------------------------------------------------------------------------
前方高能预警， 一般前端开发不需要了解那么多，不过如果你如果都看懂了，来欧朋浏览器面试吧 :)

抛开浏览器内核开销不算， 从Javascript 执行效率来讲，Virtual DOM 和DOM之间到底有多大差别呢？

这个其实可以回答Virtual DOM 到底有多快这个问题上面。 了解这个问题，需要了解浏览器内核，DOM以及网页文档到底都是什么关系。


![](/uploads/raw7.png)
很吃惊吧。



-------------------------------------------------------------------------------------------------------------
回答 @[杨伟贤](http://www.zhihu.com/people/yang-wei-xian)

你用的技巧称为是 _DocumentFragment,_ 参见
[Document Object Model (Core) Level 1](https://link.zhihu.com/?target=http%3A//www.w3.org/TR/REC-DOM-Level-1/level-one-core.html%23ID-B63ED1A3)
这个改善性能技巧早期流行其实主要是防止浏览器在JS修改DOM的时候立即Reflow。(防止你在修改了DOM以后立即取元素大小一类的）。

这个技巧在现代浏览器里面基本没有作用，因为，基本上Reflow和Layout都是能延迟就延迟。
不相信的话你这样写

<div class="highlight">

```
var tplDom = document.createElement('div');
container.appendChild(tplDom);
tplDom.innerHTML = html;
html = '';

```

</div>

tplDom先加入global DOM然后在修改，这个算是刷global DOM了吧？
在chromium里面没有任何区别的。
我更改以后的测试用例里面就是这样写的：）

所以，这个和刷不刷global DOM没有任何关系的。你看到的性能提升，其实是避免了
container.innerHTML += html;
中+=, 因为+= 是其实是一个替代操作，而不是增量操作。

另外, 你测出来的只是插入新元素时间而代码里面还需要删掉之前的节点的代码.
因为对于react.js
第一次运行时间 ＝ JIT时间+插入时间
多次以后 = 更新节点时间。 （v8优化器原因）

而更新节点 ＝ 删除节点＋插入新的节点 （你也可以用replaceChild, 性能没有能测出来差异）.

react.js由于第一次运行带着JIT,所以没有办法剥离出来纯插入时间。于是加入删除之前节点的代码，然后多次运行测试更新节点的时间才是有比较性的。

