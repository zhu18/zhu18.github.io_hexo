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

<noscript><img src="https://pic2.zhimg.com/b94c1db9c8dc910b47ef6f9fdbcb534d_b.png" data-rawwidth="517" data-rawheight="352" class="origin_image zh-lightbox-thumb" width="517" data-original="https://pic2.zhimg.com/b94c1db9c8dc910b47ef6f9fdbcb534d_r.png">这里看出问题没？</noscript>

![](https://pic2.zhimg.com/b94c1db9c8dc910b47ef6f9fdbcb534d_b.png)这里看出问题没？

整个测试用例的模式是：

1.  构造一个 String, 包含一个1000个 Tag。

2.  把这1000个 Tag 一次性添加到 DOM 树中。

其实，**题主的测试用例只做了一次 DOM 操作**。。而且主要问题是，如果你真的做一个WebAPP, 然后直接操作DOM更新，更新的模式完全不是这样子的。

在现实中，更新模式更像这个样子滴：
同样是 1000 元素需要更新， 你的界面上面分成了20个逻辑区域（或者层，或者 view, 或者whatever 框架取的名字）, **每个区域 50 个元素。在界面需要更新的时候，每个逻辑区域分别操作 DOM Tree 更新**。

那么代码看起来更像是这样子的:

<noscript><img src="https://pic2.zhimg.com/54b8f7da17021f35800e48adeee7b119_b.png" data-rawwidth="537" data-rawheight="353" class="origin_image zh-lightbox-thumb" width="537" data-original="https://pic2.zhimg.com/54b8f7da17021f35800e48adeee7b119_r.png"></noscript>

![](https://pic2.zhimg.com/54b8f7da17021f35800e48adeee7b119_b.png)
然后我们再来看看结果

<noscript><img src="https://pic1.zhimg.com/6ef03bab2d7e4612a2a8def8b0796918_b.png" data-rawwidth="640" data-rawheight="234" class="origin_image zh-lightbox-thumb" width="640" data-original="https://pic1.zhimg.com/6ef03bab2d7e4612a2a8def8b0796918_r.png"></noscript>

![](https://pic1.zhimg.com/6ef03bab2d7e4612a2a8def8b0796918_b.png)
天了撸， 发生了什么， 原生的怎么慢这么多。（React.js 并不需要修改，无论如何每个区域都是把新的操作作用在Virtual DOM上面，然后每帧只会调用一次Render)。

**2\. React.js 相对于直接操作原生DOM有很大的性能优势， 背后的技术支撑是基于virtual DOM的batching 和diff。**

原生DOM 操作慢吗？

做为一个浏览器内核开发人员， 我可以负责任的告诉你，慢。
你觉得只是改变一个字符串吗？

<noscript><img src="https://pic2.zhimg.com/96f472167b8944d42aabe61393c68241_b.jpg" data-rawwidth="500" data-rawheight="400" class="origin_image zh-lightbox-thumb" width="500" data-original="https://pic2.zhimg.com/96f472167b8944d42aabe61393c68241_r.jpg">我们来看看你在插入一个DOM元素的时候，发生了什么.</noscript>

![](https://pic2.zhimg.com/96f472167b8944d42aabe61393c68241_b.jpg)我们来看看你在插入一个DOM元素的时候，发生了什么.

<noscript><img src="https://pic1.zhimg.com/fe8b480c0016fd023b25bd4c4e8f3914_b.png" data-rawwidth="559" data-rawheight="225" class="origin_image zh-lightbox-thumb" width="559" data-original="https://pic1.zhimg.com/fe8b480c0016fd023b25bd4c4e8f3914_r.png"></noscript>

![](https://pic1.zhimg.com/fe8b480c0016fd023b25bd4c4e8f3914_b.png)
实际上，浏览器在收到更新的DOM Tree需要立即调用HTML Parser对传入的字符串进行解析，这个，呃，耗的时间可不是字符串替换可以比的哦 .

(_其实你们已经处于一个好时代了，换做几年前，浏览器还可能会花几秒到几分钟给你Reflow一下）_

这个例子还算简单的了，如果你插入的标签里面包括了脚本，浏览器可能还需要即时编译的你脚本(JIT).

_这些时间都算在你的DOM操作中的哦**。**_

我们再来看看统计。

<noscript><img src="https://pic4.zhimg.com/93799d4434552f68d7c8fcf9a39bdf6b_b.png" data-rawwidth="523" data-rawheight="237" class="origin_image zh-lightbox-thumb" width="523" data-original="https://pic4.zhimg.com/93799d4434552f68d7c8fcf9a39bdf6b_r.png"></noscript>

![](https://pic4.zhimg.com/93799d4434552f68d7c8fcf9a39bdf6b_b.png)
131 ms **都花在了Loading 里面(ParseHTML)**。

_另外注意一些细节，Profiler 报告整个函数使用了418ms, 因为有些时间在JS里面是统计不到的，比如Rendering的时间， 所以， 多用Profiler._

我们再来看一个图， 这个原测试用例的Profiling， 1000 Tag 一次插入

<noscript><img src="https://pic4.zhimg.com/3ffd05a65cdafc445226abfb79a04be3_b.png" data-rawwidth="479" data-rawheight="194" class="origin_image zh-lightbox-thumb" width="479" data-original="https://pic4.zhimg.com/3ffd05a65cdafc445226abfb79a04be3_r.png"></noscript>

![](https://pic4.zhimg.com/3ffd05a65cdafc445226abfb79a04be3_b.png)
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

－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－
前方高能预警， 一般前端开发不需要了解那么多，不过如果你如果都看懂了，来欧朋浏览器面试吧 :)

抛开浏览器内核开销不算， 从Javascript 执行效率来讲，Virtual DOM 和DOM之间到底有多大差别呢？

这个其实可以回答Virtual DOM 到底有多快这个问题上面。 了解这个问题，需要了解浏览器内核，DOM以及网页文档到底都是什么关系。

<noscript><img src="https://pic4.zhimg.com/40b07ca3f65089ee272d690dd3b10d2b_b.png" data-rawwidth="582" data-rawheight="335" class="origin_image zh-lightbox-thumb" width="582" data-original="https://pic4.zhimg.com/40b07ca3f65089ee272d690dd3b10d2b_r.png"></noscript>

![](https://pic4.zhimg.com/40b07ca3f65089ee272d690dd3b10d2b_b.png)
很吃惊吧。

其实DOM 完全不属于Javascript (也不在Javascript 引擎中存在). Javascript 其实是一个非常独立的引擎，DOM其实是浏览器引出的一组让Javascript操作HTML文档的API而已。

_如果你了解Java的话，这个相当于JNI._

那么问题就来了, **DOM的操作对于Javascript实际上是外部函数调用**。 每次Javascript 调用DOM时候，都需要保存当前的上下文，然后调用DOM, 然后再恢复上下文。 这个在没有即时编译(JIT)的时代可能还好点。 自从加入即时编译(JIT), 这种上下文开销简直就是。。惨不忍睹。

我们来顺便写一段Javascript.

<noscript><img src="https://pic2.zhimg.com/db43a7ae9e45c53c7246c3839d4a4ee5_b.png" data-rawwidth="408" data-rawheight="228" class="content_image" width="408"></noscript>

![](https://pic2.zhimg.com/db43a7ae9e45c53c7246c3839d4a4ee5_b.png)
然后在v8里面跑一跑(直接使用v8 sample中的shell ). 由于v8 单独的Shell中不存在DOM， 我们用print代替， print 是外部函数， 调用它和调用DOM是一回事。

调用的堆栈看起来是这样的。

<noscript><img src="https://pic4.zhimg.com/310b77624852fffe1e2160d7acfed8af_b.png" data-rawwidth="570" data-rawheight="236" class="origin_image zh-lightbox-thumb" width="570" data-original="https://pic4.zhimg.com/310b77624852fffe1e2160d7acfed8af_r.png"></noscript>

![](https://pic4.zhimg.com/310b77624852fffe1e2160d7acfed8af_b.png)
这里可以看到V8是如何执行JIT代码，然后JIT代码调用到Print的过程 (JIT 代码就是没有符号的那一堆，Frame #1 - #5.）

我们来看看v8 JIT 生成的代码.

<noscript><img src="https://pic4.zhimg.com/87d461d6e5f85a11cb6a99d78a2c0a9b_b.png" data-rawwidth="583" data-rawheight="128" class="origin_image zh-lightbox-thumb" width="583" data-original="https://pic4.zhimg.com/87d461d6e5f85a11cb6a99d78a2c0a9b_r.png"></noscript>

![](https://pic4.zhimg.com/87d461d6e5f85a11cb6a99d78a2c0a9b_b.png)
看到这里还算合理， 一个call调走，不过我们来看看 CallApiAccessorStub是个什么鬼：

<noscript><img src="https://pic3.zhimg.com/3ad42d89d25a6fc48b3965ad88399892_b.png" data-rawwidth="257" data-rawheight="261" class="content_image" width="257"></noscript>

![](https://pic3.zhimg.com/3ad42d89d25a6fc48b3965ad88399892_b.png)
60+ 条额外指令用于保存上下文和堆栈检查。。
我靠，我就调个函数，你至于吗..

_当然，现代的JIT技术已经进步很多了，换到几年前，这个函数直接就不JIT了 (不编译了, 在V8中即不优化，你懂的，慢10到100倍）._

而Virtual DOM的执行完全都在Javascript 引擎中，完全不会有这个开销。

什么，你说我第一个测试结果里面两边的速度就差一半嘛（117 ms vs 235 ms)，react.js还只是做了一次DOM操作，原生的可是做了50次哦， 你说的virtual DOM框架完全不会有开销是几个意思？

我们来稍微改改测试代码。

<noscript><img src="https://pic3.zhimg.com/50c0b736b5ae63b0c35c925670ec537e_b.png" data-rawwidth="576" data-rawheight="315" class="origin_image zh-lightbox-thumb" width="576" data-original="https://pic3.zhimg.com/50c0b736b5ae63b0c35c925670ec537e_r.png">给你们每个条目加个标示符，这样每次更新的DOM都不一样，我看React.js你怎么做Diff, 哇哈哈哈哈哈.</noscript>

![](https://pic3.zhimg.com/50c0b736b5ae63b0c35c925670ec537e_b.png)给你们每个条目加个标示符，这样每次更新的DOM都不一样，我看React.js你怎么做Diff, 哇哈哈哈哈哈.

然后你可以多点几次React的测试按钮，一般来说来，第二次以后, 你就可以看到性能稳定在这个数字。

<noscript><img src="https://pic4.zhimg.com/88388ee7e8345d256bed0ee90a942083_b.png" data-rawwidth="638" data-rawheight="359" class="origin_image zh-lightbox-thumb" width="638" data-original="https://pic4.zhimg.com/88388ee7e8345d256bed0ee90a942083_r.png"></noscript>

![](https://pic4.zhimg.com/88388ee7e8345d256bed0ee90a942083_b.png)
这个是个什么情况？ 这个可不是React.js Diff的功劳哦？因为我们每次的更新都是完全不同的，它木有办法占便宜做Diff哦。。

这个其实是Javascript 引擎的工作特性引起。Javascript 引擎目前用的都是即时编译(JIT). 所以

*   _**第一次点击运行的时候所耗的时间 ＝ 框架被编译的时间(JIT) + 执行时间**_

*   _**之后执行的时间 ＝ 执行时间。**_

所以, **53 ms那个才是Virtual DOM 的真实执行性能哦, 是不是觉得好快呀**。

_当然, v8的JIT方法还要特殊一些, 他是两次编译的, 第一次粗编译，第二次会把执行很多的函数进行精细编译. 所以第三次以后才是最快的时间。_
_
两次编译虽然很有趣也很有用，鉴于这个帖子实在是太长了，这里就不讲了。 有兴趣看这个:_

[v8: a tale of two compilers -- wingolog](https://link.zhihu.com/?target=https%3A//wingolog.org/archives/2011/07/05/v8-a-tale-of-two-compilers)

_原测试用例在测react第二次运行的时候会很慢（大概4s左右), 原因是这个：_
onClick: this.select.bind(null, this.props.data[i])

bind 会每次创建一个新的函数对象，于是原测试里面每次点击会创建1000个新的函数对象。恭喜原作者，JS的内存真是不要钱。。
我的测试用例里面暂时去掉了，彻底修复可以不用bind, 指向一个函数即可，然后用其他方法为每个列表项保存状态。

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

修改后的测试用例在
[<span class="invisible">http://</span><span class="visible">fredrikluo.github.io/te</span><span class="invisible">stcases/reactjstest-1.html</span><span class="ellipsis"></span>](https://link.zhihu.com/?target=http%3A//fredrikluo.github.io/testcases/reactjstest-1.html)
你可以试试，这个是我这里测出来的结果。

<noscript><img src="https://pic4.zhimg.com/8bb3f203a2c082ef6b1e821d7d7d9a9f_b.png" data-rawwidth="597" data-rawheight="192" class="origin_image zh-lightbox-thumb" width="597" data-original="https://pic4.zhimg.com/8bb3f203a2c082ef6b1e821d7d7d9a9f_r.png"></noscript>

![](https://pic4.zhimg.com/8bb3f203a2c082ef6b1e821d7d7d9a9f_b.png)
很接近了是不是？不可否认这是一个非常好的优化，
是不是觉得React已经没有什么优势？

少年，你太天真了，浏览器这个世界很险恶的，我刚才说Layout和Reflow被怎么了来着？
被延迟了。。

我们来看看profiler的输出。
React 的性能统计

<noscript><img src="https://pic3.zhimg.com/eae5800183ba03b92c54182f0e3cf666_b.png" data-rawwidth="518" data-rawheight="235" class="origin_image zh-lightbox-thumb" width="518" data-original="https://pic3.zhimg.com/eae5800183ba03b92c54182f0e3cf666_r.png">Raw的性能统计</noscript>

![](https://pic3.zhimg.com/eae5800183ba03b92c54182f0e3cf666_b.png)Raw的性能统计

<noscript><img src="https://pic2.zhimg.com/372dd6baf138b3a93ee58a1ccaada0d5_b.png" data-rawwidth="524" data-rawheight="234" class="origin_image zh-lightbox-thumb" width="524" data-original="https://pic2.zhimg.com/372dd6baf138b3a93ee58a1ccaada0d5_r.png"></noscript>

![](https://pic2.zhimg.com/372dd6baf138b3a93ee58a1ccaada0d5_b.png)
事实上React的耗用的时间的是80m, 而优化后的Raw也有144 ms, 接近一半的时间。

**为蜀莫呢？**

有个31ms 和95ms的rendering差别，而在这个例子里面， rendering = Recalculate Style 和Layout（参见上面的堆栈，或者你自己也可以试试）。而这个差异，在JS里面是测不到了，因为， 呃，他们被延迟了。。

原因也很简单，React 修改的粒度更小。

virtual DOM每次用Batch + diff来修改DOM的时候, 假如你有个<span class=''123"> abc</span>如果只是内容变了，那spanNode.textContent会被换掉, style啥的不会动。 而在原生例子里面，整个spanNode被换掉。

_浏览器： 你把整个节点都换了，我肯定只能重新计算style和layout了， 怪我咯。_

从本质上面讲， react.js 和自己写算法直接操作原生的DOM是这样一个关系：

**修改的内容 -> React.js -> DOM操作**
**修改的内容 -> 你的算法 -> DOM操作**

**React.js 和你的算法最后都是把特定的意图翻译成一组DOM操作，差别只是谁的翻译更加优化而已**

而原回答其实想说明的时候，react.js 在使用virtual DOM这套算法以后DOM操作在通常情况下比自己的写是优化非常多的。这个其实是对“使用react.js和（用自己算法）操作DOM哪个快”这个问题的直接回答。 而你当然可以进一步优化算法，在特定的环境下面接近或者超过React, 不过，这个在实际开发中并没有适普性。

这个其实和问编译器和手动汇编哪个快是一模一样

**算法 -> 原代码 ->编译器－>汇编**
**算法 ->手动翻译->汇编**

而在目前CPU的复杂程度下，手动翻译反而大部分时间比不上编译器， 复杂度越高，需要考虑的变量越多，越容易用算法来实现而人脑总是爱忘东西的。

不可否认是原测试用例里面的不管是测raw还是测react的的算法都写得很有问题。所以对原回答中只是尽量不做大的修改来说明问题而已。

最后还要感谢你的提问，否则没法讲得这个深度，其实在写virtual DOM JIT那一段的时候我就很犹豫是不是讲得有点过了。
-----------------------------------------------------------------------------------------------------------
回答<span><span data-reactroot="" class="UserLink">

<div class="Popover">

<div id="Popover-92709-47582-toggle" aria-haspopup="true" aria-expanded="false" aria-owns="Popover-92709-47582-content">[@尤雨溪](/people/cfdec6226ece879d2571fbc274372e9f)</div>

</div>

</span></span>
呃。。你说的这句话是吧 － “发明出来就是为了提高性能” 以及 “没有那么多哲学方面的东西需要承载”。
既然我们讨论的是“发明出来”， 我们就来看看原团队的意思咯， 戳这里， 这个原团队的blog post：
[Why did we build React?](https://link.zhihu.com/?target=https%3A//facebook.github.io/react/blog/2013/06/05/why-react.html)
－－ 摘要如下：
**Why did we build React?**
There are a lot of JavaScript MVC frameworks out there. Why did we build React and why would you want to use it?
**React isn't an MVC framework.**
....
**React doesn't use templates.**
...
**Reactive updates are dead simple**
React really shines when your data changes over time.
....
Because this re-render is so fast (around 1ms for TodoMVC), the developer doesn't need to explicitly specify data bindings. We've found this approach makes it easier to build apps
....
这个才是全文重点, 请再次默念下“**really shines**”

这篇Post翻译成大白话就是：
**哥做了一个框架， 有这个功能，还有这个功能，但是最牛逼的功能就是：你的数据变的时候渲染的特别快哦, 快得你们爱怎么搞怎么搞。**

真的不是性能嘛？...

PS: 虽然我觉得你说得很有道理，他们都是react.js的好处啦，不过就从"why"来说

*   但是“通过引入函数式思维来加强对状态的管理”， 貌似木有在“why”里面提到？

*   第三方那个我木有找到引用。
*   而“Virtual DOM 可以渲染到非 DOM 的后端从而催生 ReactNative”, 是在：“Because React has its own lightweight representation of the document, we can do some pretty cool things with it:”

听起来貌似是：
“React really shines when your data changes over time.

In a traditional JavaScript application, you need to look at what data changed and imperatively make changes to the DOM to keep it up-to-date. Even AngularJS, which provides a declarative interface via directives and data binding [requires a linking function to manually update DOM nodes](https://link.zhihu.com/?target=https%3A//code.angularjs.org/1.0.8/docs/guide/directive%23reasonsbehindthecompilelinkseparation).

React takes a different approach.”
