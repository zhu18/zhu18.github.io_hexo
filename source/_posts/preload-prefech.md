---
title: 深入研究Chrome：Preload与Prefetch原理
date: 2017-07-18 16:23:53
tags: [性能, chrome]
---

<div class="rich_media_content " id="js_content">

<section style="color: rgb(63, 63, 63); font-size: 14px; font-family: Avenir, -apple-system-font, 微软雅黑, sans-serif; text-align: left; box-sizing: border-box;">

<section class="" style="box-sizing: border-box; text-align: left;">![](http://mmbiz.qpic.cn/mmbiz_png/uMh5nccSicmLhOGj31Ox7URHHfI7myYCg3J2TvLkpVQYfcmTpOqxuS8WeHW7lD3Ljm3JiaXHHFMxnjaqhAWLcX4Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)</section>

<section class="" style="box-sizing: border-box; color: rgb(145, 145, 145); text-align: left; line-height: 1em; margin-top: 13px; padding-left: 14px;">译者 | gy134340</section>

<section class="" style="box-sizing: border-box; color: rgb(145, 145, 145); text-align: left; line-height: 1em; margin-top: 13px; padding-left: 14px;">校对者 | IridescentMia,vuuihc</section>

<section class="" style="box-sizing: border-box; font-size: 15px; text-align: justify; line-height: 27px; color: rgb(89, 89, 89); background-color: rgb(239, 239, 239); padding: 19px; margin-top: 40px; margin-right: 8px; margin-left: 8px;">今天我们来深入研究一下 Chrome 的网络协议栈，来更清晰的描述早期网络加载（像 <link rel=“preload”> 和 <link rel=“prefetch”><a style="color: #5baceb; word-break: break-all;"></a>）背后的工作原理，让你对其更加了解。</section>

**像其他文章描述的那样，preload 是声明式的 fetch，可以强制浏览器请求资源，同时不阻塞文档 onload 事件。**

**Prefetch 提示浏览器这个资源将来可能需要**，但是把决定是否和什么时间加载这个资源的决定权交给浏览器。


<!-- more -->


<a style="color: #5baceb; word-break: break-all;">![](http://mmbiz.qpic.cn/mmbiz_jpg/uMh5nccSicmLhOGj31Ox7URHHfI7myYCgFkd0YxGIZBVP0brP2MQd7YDHcrP1Avpee2LN3iasMwXcu1ZgsTZxrUA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1)</a>

Preload 将 load 事件与脚本解析过程解耦，如果你还没有用过它，看看 Yoav Weiss 的文章 <a style="color: #5baceb; word-break: break-all;">Preload: What is it Good For?</a>（https://www.smashingmagazine.com/2016/02/preload-what-is-it-good-for/）。

<section class="" style="box-sizing: border-box; font-size: 20px; text-align: center;">**<span style="height: 65px; line-height: 95px; color: rgb(60, 112, 198); border-bottom: 2px solid rgb(27, 95, 160); background-image: url(&quot;http://mmbiz.qpic.cn/mmbiz_jpg/uMh5nccSicmLhOGj31Ox7URHHfI7myYCgvMjjiaiar3k8SjCwI5BbtTPGibCXTGI9qV4vTT4KwRRj3u0AmeNPyoApw/0?wx_fmt=jpeg&quot;); background-position: 50% 50%; background-repeat: no-repeat; background-attachment: initial; background-origin: initial; background-clip: initial; background-size: 92px; margin-top: 38px; margin-bottom: 10px; padding-bottom: 7px; display: inline-block; font-size: 16px;">Preload 在生产环境的成功案例</span>**</section>

在我们深入细节之前，下面是上一年发现的使用 proload 并对加载产生积极影响的案例总结。

<section class="" style="box-sizing: border-box; font-size: 16px; text-align: center;"><span style="display: inline-block; height: 38px; line-height: 42px; color: rgb(60, 112, 198); background-image: url(&quot;http://mmbiz.qpic.cn/mmbiz_jpg/uMh5nccSicmLhOGj31Ox7URHHfI7myYCgvMjjiaiar3k8SjCwI5BbtTPGibCXTGI9qV4vTT4KwRRj3u0AmeNPyoApw/0?wx_fmt=jpeg&quot;); background-position: left center; background-repeat: no-repeat; background-attachment: initial; background-origin: initial; background-clip: initial; background-size: 63px; padding-left: 35px; margin-top: 38px; margin-bottom: 10px;">案例一：Housing.com的案例</span></section>

Housing.com 在对他们的渐进式 Web 应用程序的脚本转用 proload 看到<a style="color: #5baceb; word-break: break-all;">**大约缩短了10%的可交互时间**</a>。

<a style="color: #5baceb; word-break: break-all;">![](http://mmbiz.qpic.cn/mmbiz_png/uMh5nccSicmLhOGj31Ox7URHHfI7myYCgKC7JZD5p6YKhbY6KmWF0vIPg4nPn0GPjJjIORq1S5FJf5zzYrI19icA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)</a>

Shopify 在转用 <a style="color: #5baceb; word-break: break-all;">preload 加载字体</a>后在 Chrome 桌面版获得了 <a style="color: #5baceb; word-break: break-all;">**50%**（1.2s）</a> 的文字渲染优化，这完全解决了他们的文字闪动问题。

<a style="color: #5baceb; word-break: break-all;">![](http://mmbiz.qpic.cn/mmbiz_png/uMh5nccSicmLhOGj31Ox7URHHfI7myYCgKgZdsHdRU7Y66tBPK5B1hsficUXJI1mcrJ5Evo4EQTECC92IqIdCw4A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)</a>

左边：使用 preload，右边：不使用 preload（<a style="color: #5baceb; word-break: break-all;">视频</a>）

<a style="color: #5baceb; word-break: break-all;">![](http://mmbiz.qpic.cn/mmbiz_png/uMh5nccSicmLhOGj31Ox7URHHfI7myYCgLqAYpjvERaAiasG86xJvMpPNibwibbk0MyGZZezzNXfYLvHLxvfe4lnvQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)</a>

使用`<link rel=”preload”>` 加载字体。

<section class="" style="box-sizing: border-box; font-size: 16px; text-align: center;"><span style="display: inline-block; height: 38px; line-height: 42px; color: rgb(60, 112, 198); background-image: url(&quot;http://mmbiz.qpic.cn/mmbiz_jpg/uMh5nccSicmLhOGj31Ox7URHHfI7myYCgvMjjiaiar3k8SjCwI5BbtTPGibCXTGI9qV4vTT4KwRRj3u0AmeNPyoApw/0?wx_fmt=jpeg&quot;); background-position: left center; background-repeat: no-repeat; background-attachment: initial; background-origin: initial; background-clip: initial; background-size: 63px; padding-left: 35px; margin-top: 38px; margin-bottom: 10px;">案例二：Treebo</span></section>

Treebo，印度最大的旅馆网站之一，在 3G 网络下对其桌面版试验，在对其顶部图片和主要的 Webpack 打包文件使用 preload 之后，在**首屏绘制和可交互延迟分别减少了** <a style="color: #5baceb; word-break: break-all;">**1s**</a>。

<a style="color: #5baceb; word-break: break-all;">![](http://mmbiz.qpic.cn/mmbiz_png/uMh5nccSicmLhOGj31Ox7URHHfI7myYCgYgkfVBqLTuweiaZ792oQn0CvsJ4y1u0t0OoLLL7gXTOAnTCMCic7QWgA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)</a>

同样的，在对自己的渐进式 Web 应用程序主要打包文件使用 preload 之后，Flipkart 在路由解析之前 **节省了大量的主线程空闲时间**（在 3G 网络下的低性能手机下）。

<a style="color: #5baceb; word-break: break-all;">![](http://mmbiz.qpic.cn/mmbiz_png/uMh5nccSicmLhOGj31Ox7URHHfI7myYCgXjAcP4gB2UThU8S83vib1Cw09ib8b3iaV4FULj6sGQ7l9x3ckMNXS4cBg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)</a>

上面：未使用 preload，下面：使用 preload

Chrome 数据保护团队在对脚本和 CSS 样式表使用 preload 之后，发现页面首次绘制时间获得<a style="color: #5baceb; word-break: break-all;">**平均 12%**</a> 的速度提升。

对于 prefetch ，它被广泛使用，在 Google 我们仍用它来获取可以加快 <a style="color: #5baceb; word-break: break-all;">搜索结果页面</a> 的渲染的关键资源。

Preload 在很多大型网站都有实际应用，这点你在接下来的文章里也可以看到，让我们来仔细探讨下网络协议栈实际上是如何对待 preload 和 prefetch 的。

<section class="" style="box-sizing: border-box; font-size: 20px; text-align: center;">**<span style="height: 65px; line-height: 95px; color: rgb(60, 112, 198); border-bottom: 2px solid rgb(27, 95, 160); background-image: url(&quot;http://mmbiz.qpic.cn/mmbiz_jpg/uMh5nccSicmLhOGj31Ox7URHHfI7myYCgvMjjiaiar3k8SjCwI5BbtTPGibCXTGI9qV4vTT4KwRRj3u0AmeNPyoApw/0?wx_fmt=jpeg&quot;); background-position: 50% 50%; background-repeat: no-repeat; background-attachment: initial; background-origin: initial; background-clip: initial; background-size: 92px; margin-top: 38px; margin-bottom: 10px; padding-bottom: 7px; display: inline-block; font-size: 16px;">网络协议栈如何对待 preload 和 prefetch？</span>**</section>

<section class="" style="box-sizing: border-box; font-size: 16px; text-align: left; margin-top: 30px; margin-left: 8px; color: rgb(60, 112, 198);"><span style="display: inline-block; width: 15px; height: 15px; margin-right: 10px; background-image: url(&quot;http://mmbiz.qpic.cn/mmbiz_png/uMh5nccSicmLhOGj31Ox7URHHfI7myYCgHnPNoGLcfCtRCsGgZB6ytGYqXVfn99IqJqo77u8icvnzZic7ibiaZoLPtA/0?wx_fmt=png&quot;); background-position: center center; background-repeat: no-repeat; background-attachment: initial; background-origin: initial; background-clip: initial; background-size: 100% 100%;"></span>什么时候又该用  <link rel=”prefetch”> ?</section>

**建议：对于当前页面很有必要的资源使用 preload，对于可能在将来的页面中使用的资源使用 prefetch。**

preload 是对浏览器指示预先请求当前页需要的资源（关键的脚本，字体，主要图片）。

prefetch 应用场景稍微又些不同 —— 用户将来可能在其他部分（比如视图或页面）使用到的资源。如果 A 页面发起一个 B 页面的 prefetch 请求，这个资源获取过程和导航请求可能是同步进行的，而如果我们用 preload 的话，页面 A 离开时它会立即停止。

使用 preload 和 prefetch，我们有了对当前页面和将来页面加载关键资源的解决办法。

<section class="" style="box-sizing: border-box; font-size: 16px; text-align: left; margin-top: 30px; margin-left: 8px; color: rgb(60, 112, 198);"><span style="display: inline-block; width: 15px; height: 15px; margin-right: 10px; background-image: url(&quot;http://mmbiz.qpic.cn/mmbiz_png/uMh5nccSicmLhOGj31Ox7URHHfI7myYCgHnPNoGLcfCtRCsGgZB6ytGYqXVfn99IqJqo77u8icvnzZic7ibiaZoLPtA/0?wx_fmt=png&quot;); background-position: center center; background-repeat: no-repeat; background-attachment: initial; background-origin: initial; background-clip: initial; background-size: 100% 100%;"></span> <link rel="preload"> 和 <link rel="prefetch"> 的缓存行为</section>

<a style="color: #5baceb; word-break: break-all;">Chrome 有四种缓存</a>: HTTP 缓存，内存缓存，Service Worker 缓存和 Push 缓存。preload 和 prefetch 都被存储在 **HTTP 缓存中**。

当一个资源被 **preload 或者 prefetch** 获取后，它可以从 HTTP 缓存移动至渲染器的内存缓存中。如果资源可以被缓存（比如说存在有效的<a style="color: #5baceb; word-break: break-all;">cache-control</a> 和 max-age），它被存储在 HTTP 缓存中可以被**现在或将来的任务使用**，如果资源不能被缓存在 HTTP 缓存中，作为代替，它被放在内存缓存中直到被使用。

<section class="" style="box-sizing: border-box; font-size: 16px; text-align: left; margin-top: 30px; margin-left: 8px; color: rgb(60, 112, 198);"><span style="display: inline-block; width: 15px; height: 15px; margin-right: 10px; background-image: url(&quot;http://mmbiz.qpic.cn/mmbiz_png/uMh5nccSicmLhOGj31Ox7URHHfI7myYCgHnPNoGLcfCtRCsGgZB6ytGYqXVfn99IqJqo77u8icvnzZic7ibiaZoLPtA/0?wx_fmt=png&quot;); background-position: center center; background-repeat: no-repeat; background-attachment: initial; background-origin: initial; background-clip: initial; background-size: 100% 100%;"></span>Chrome 对于 preload 和 prefetch 的网络优先级如何？</section>

下面是在 Blink 内核的 Chrome 46 及更高版本中不同资源的加载优先级情况（ <a style="color: #5baceb; word-break: break-all;">Pat Meenan</a>）

<a style="color: #5baceb; word-break: break-all;">![](http://mmbiz.qpic.cn/mmbiz_jpg/uMh5nccSicmLhOGj31Ox7URHHfI7myYCgU5BTXFZZf6AI0xNuVW9ys5sJ1WbcALnGgIKfJ018b8Gia3VlG4MqesA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1)</a>

preload 用 “as” 或者用 “type” 属性来表示他们请求资源的优先级（比如说 preload 使用 as="style" 属性将获得最高的优先级）。没有 “as” 属性的将被看作异步请求，“Early”意味着在所有未被预加载的图片请求之前被请求（“late”意味着之后）,感谢 Paul Irish 更新这张关于开发者工具以及网络层上各种请求优先级的表。

我们来谈一下这张表。

**脚本根据它们在文件中的位置是否异步、延迟或阻塞获得不同的优先级**：

*   网络在第一个图片资源之前阻塞的脚本在网络优先级中是中级

*   网络在第一个图片资源之后阻塞的脚本在网络优先级中是低级

*   异步／延迟／插入的脚本（无论在什么位置）在网络优先级中是很低级

图片（视口可见）将会获得相对于视口不可见图片（低级）的更高的优先级（中级），所以某些程度上 Chrome 将会尽量懒加载这些图片。低优先级的图片在布局完成被视口发现时，将会获得优先级提升（但是注意已经在布局完成后的图片将不会更改优先级）。

preload 使用 “as” 属性加载的资源将会获得与资源 “type” 属性所拥有的**相同的优先级**。比如说，preload as="style" 将会获得比 as=“script” 更高的优先级。这些资源同样会受内容安全策略的影响（比如说，脚本会受到其 “src” 属性的影响）。

不带 “as” 属性的 preload 的优先级将会等同于异步请求。

如果你想了解各种资源加载时的优先级属性，从开发者工具的 Timeline/Performance 区域的 Network 区域都能看到相关信息：

<a style="color: #5baceb; word-break: break-all;">![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)</a>

在 Network 面板下的“Priority”部分：

<a style="color: #5baceb; word-break: break-all;">![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)</a>

<section class="" style="box-sizing: border-box; font-size: 16px; text-align: left; margin-top: 30px; margin-left: 8px; color: rgb(60, 112, 198);"><span style="display: inline-block; width: 15px; height: 15px; margin-right: 10px; background-image: url(&quot;http://mmbiz.qpic.cn/mmbiz_png/uMh5nccSicmLhOGj31Ox7URHHfI7myYCgHnPNoGLcfCtRCsGgZB6ytGYqXVfn99IqJqo77u8icvnzZic7ibiaZoLPtA/0?wx_fmt=png&quot;); background-position: center center; background-repeat: no-repeat; background-attachment: initial; background-origin: initial; background-clip: initial; background-size: 100% 100%;"></span>当页面 preload 已经在 Service Worker 缓存及 HTTP 缓存中的资源时会发生什么？</section>

这就要说看情况了，但通常来说，会是比较好的情况 —— 如果资源没有超出 HTTP 缓存时间或者 Service Worker 没有主动重新发起请求，那么浏览器就不会再去请求这个资源了。

如果资源在 HTTP 缓存（ Service Worker 缓存和网络中），那么 preload 将会获得一次缓存命中。

<section class="" style="box-sizing: border-box; font-size: 16px; text-align: left; margin-top: 30px; margin-left: 8px; color: rgb(60, 112, 198);"><span style="display: inline-block; width: 15px; height: 15px; margin-right: 10px; background-image: url(&quot;http://mmbiz.qpic.cn/mmbiz_png/uMh5nccSicmLhOGj31Ox7URHHfI7myYCgHnPNoGLcfCtRCsGgZB6ytGYqXVfn99IqJqo77u8icvnzZic7ibiaZoLPtA/0?wx_fmt=png&quot;); background-position: center center; background-repeat: no-repeat; background-attachment: initial; background-origin: initial; background-clip: initial; background-size: 100% 100%;"></span>这将会浪费用户的带宽吗？</section>

**用“preload”和“prefetch”情况下，如果资源不能被缓存，那么都有可能浪费一部分带宽。**

没有用到的 preload 资源在 Chrome 的 console 里会在 onload 事件 3s 后发生警告。

<a style="color: #5baceb; word-break: break-all;">![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)</a>

原因是你可能为了改善性能使用 preload 来缓存一定的资源，但是如果没有用到，你就做了无用功。在手机上，这相当于浪费了用户的流量，所以明确你要 preload 对象。

<section class="" style="box-sizing: border-box; font-size: 16px; text-align: left; margin-top: 30px; margin-left: 8px; color: rgb(60, 112, 198);"><span style="display: inline-block; width: 15px; height: 15px; margin-right: 10px; background-image: url(&quot;http://mmbiz.qpic.cn/mmbiz_png/uMh5nccSicmLhOGj31Ox7URHHfI7myYCgHnPNoGLcfCtRCsGgZB6ytGYqXVfn99IqJqo77u8icvnzZic7ibiaZoLPtA/0?wx_fmt=png&quot;); background-position: center center; background-repeat: no-repeat; background-attachment: initial; background-origin: initial; background-clip: initial; background-size: 100% 100%;"></span>什么情况会导致二次获取？</section>

preload 和 prefetch 是很简单的工具，你很容易不小心<a style="color: #5baceb; word-break: break-all;">二次获取</a>。

不要用 “prefetch” 作为 “preload” 的后备，它们适用于不同的场景，常常会导致不符合预期的二次获取。使用 preload 来获取当前需要任务否则使用 prefetch 来获取将来的任务，不要一起用。

<a style="color: #5baceb; word-break: break-all;">![](http://mmbiz.qpic.cn/mmbiz_png/uMh5nccSicmLhOGj31Ox7URHHfI7myYCgcLVoazY07v1mKPOJyicxKU0Ju3nBphjsjSliaphkjYkHJvL0YqH4nxkw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)</a>

不要指望 preload 和 fetch() 配合使用，在 Chrome 中这样使用将会导致二次的下载。这并不只发生在异步请求的情况我们有一个关于这个问题公开的 <a style="color: #5baceb; word-break: break-all;">bug</a>。

**对 preload 使用 “as” 属性，不然将不会从中获益**。

如果你对你所 preload 的资源使用明确的 “as” 属性，比如说，脚本，你将会导致<a style="color: #5baceb; word-break: break-all;">二次获取</a>。

**preload 字体不带 crossorigin 也将会二次获取！** 确保你对 preload 的字体添加 <a style="color: #5baceb; word-break: break-all;">crossorigin</a> 属性，否则他会被下载两次，这个请求使用匿名的跨域模式。这个建议也适用于字体文件在相同域名下，也适用于其他域名的获取(比如说默认的异步获取)。

_<span style="float: left; margin-left: -32px; display: block; width: 19px; height: 17px; margin-top: 5px; background-image: url(&quot;http://mmbiz.qpic.cn/mmbiz_png/uMh5nccSicmLhOGj31Ox7URHHfI7myYCg0OwrV2ibEhEk5micGFIn8smthLxiaOIsgsg0GbmsotHbauTsSuU1HgIcQ/0?wx_fmt=png&quot;); background-position: center center; background-repeat: no-repeat; background-attachment: initial; background-origin: initial; background-clip: initial; background-size: 100% 100%;"></span>存在 intergrity 属性的资源不能使用 preload 属性（目前）也会导致二次获取。_

<span style="font-size: 16px;">链接元素的 integrity 属性目前还没有被支持，目前有一个关于它的开放</span><a style="color: rgb(91, 172, 235); word-break: break-all; font-size: 16px; text-decoration: underline;"><span style="font-size: 16px;">issue</span></a><span style="font-size: 16px;">，这意味着存在 integrity 的元素将会丢弃 preload 的资源。往宽了讲，这会导致重复的请求，你需要在安全和性能之间作出权衡。</span>

最后，虽然它不会导致二次获取，还是有下面的建议：

**不要 preload 所有东西！** 作为替代的，用 preload 来告诉浏览器一些本来不能被提早发现的资源，以便提早获取它们。

<section class="" style="box-sizing: border-box; font-size: 16px; text-align: left; margin-top: 30px; margin-left: 8px; color: rgb(60, 112, 198);"><span style="display: inline-block; width: 15px; height: 15px; margin-right: 10px; background-image: url(&quot;http://mmbiz.qpic.cn/mmbiz_png/uMh5nccSicmLhOGj31Ox7URHHfI7myYCgHnPNoGLcfCtRCsGgZB6ytGYqXVfn99IqJqo77u8icvnzZic7ibiaZoLPtA/0?wx_fmt=png&quot;); background-position: center center; background-repeat: no-repeat; background-attachment: initial; background-origin: initial; background-clip: initial; background-size: 100% 100%;"></span>我应当在页面头部加载所有的资源文件吗？有什么建议比如说限制“只加载6个文件”？</section>

这是**工具而不是规则**的好例子。你 preload 的文件数量取决于加载其他资源时网络内容、用户的带宽和其他网络状况。

尽早 preload 页面中可能需要的文件，对于脚本文件，preload 关键打包文件很有用因为它将加载与执行分离开来，`script async`不好因为它会阻塞 window 的 onload 事件。你可以尽早加载图片、样式、字体和媒体资源。大部分的 —— 最重要的是，你作为作者是可以清晰的知道哪些东西是页面目前需要的。

<section class="" style="box-sizing: border-box; font-size: 16px; text-align: left; margin-top: 30px; margin-left: 8px; color: rgb(60, 112, 198);"><span style="display: inline-block; width: 15px; height: 15px; margin-right: 10px; background-image: url(&quot;http://mmbiz.qpic.cn/mmbiz_png/uMh5nccSicmLhOGj31Ox7URHHfI7myYCgHnPNoGLcfCtRCsGgZB6ytGYqXVfn99IqJqo77u8icvnzZic7ibiaZoLPtA/0?wx_fmt=png&quot;); background-position: center center; background-repeat: no-repeat; background-attachment: initial; background-origin: initial; background-clip: initial; background-size: 100% 100%;"></span>prefetch 有哪些你需要知道的魔法属性吗？当然有！</section>

在 Chrome 中，如果用户从一个页面跳转到另一个页面，prefetch 发起的请求仍会进行不会中断。

另外，prefetch 的资源在网络堆栈中至少缓存 5 分钟，无论它是不是可以缓存的。

<section class="" style="box-sizing: border-box; font-size: 16px; text-align: left; margin-top: 30px; margin-left: 8px; color: rgb(60, 112, 198);"><span style="display: inline-block; width: 15px; height: 15px; margin-right: 10px; background-image: url(&quot;http://mmbiz.qpic.cn/mmbiz_png/uMh5nccSicmLhOGj31Ox7URHHfI7myYCgHnPNoGLcfCtRCsGgZB6ytGYqXVfn99IqJqo77u8icvnzZic7ibiaZoLPtA/0?wx_fmt=png&quot;); background-position: center center; background-repeat: no-repeat; background-attachment: initial; background-origin: initial; background-clip: initial; background-size: 100% 100%;"></span>我在 JS 中使用自定义的 “preload”，它跟原本的 rel="preload" 或者 preload 头部有什么不同？</section>

preload 将资源获取与执行解耦，像这样，preload 在标记中声明以被 Chrome preload 扫描器扫描。这意味着，在许多案例中，在 HTML 解析器获取到标签之前，preload 就会被获取（用它声明的优先级）。这将会比自定义的 preload 更加强大。

<section class="" style="box-sizing: border-box; font-size: 16px; text-align: left; margin-top: 30px; margin-left: 8px; color: rgb(60, 112, 198);"><span style="display: inline-block; width: 15px; height: 15px; margin-right: 10px; background-image: url(&quot;http://mmbiz.qpic.cn/mmbiz_png/uMh5nccSicmLhOGj31Ox7URHHfI7myYCgHnPNoGLcfCtRCsGgZB6ytGYqXVfn99IqJqo77u8icvnzZic7ibiaZoLPtA/0?wx_fmt=png&quot;); background-position: center center; background-repeat: no-repeat; background-attachment: initial; background-origin: initial; background-clip: initial; background-size: 100% 100%;"></span>等一下，我不是可以用 HTTP/2 的服务器推送来代替 preload 吗？</section>

当你知道资源加载的正确顺序时使用推送，用 service worker 来拦截那些可能需要会导致二次获取的资源请求，用 preload 来加快第一个请求的开始时间 —— 这对所有的资源获取都有用。

再次说一下，这都要看<a style="color: #5baceb; word-break: break-all;">情况</a>，我们试想一下位 Google Play 商店做购物车，对于一个向购物车的请求：

用 preload 来加载页面的主要的模块需要浏览器等待 play.google.com/cart 有效载荷以便 preload 扫描器发现依赖，但这之后会浸透网络管道可以更好的像资源发起请求，这可能不是最理想的冷启动，但对于高速缓存和带宽的后续请求非常友好。

使用 HTTP/2 的服务器推送，当请求 play.google.com/cart 我们可以快速浸透网络管道，但如果资源已经在 HTTP 或者 Service Worker 缓存中的话我们就浪费了带宽，两种方法都需要做出权衡。

虽然推送很有效，但它不像 preload 那样对所有的情况都适应。

preload 利于下载与执行的解耦，多亏其对文档 onload 事件的支持我们现在可以控制其加载完毕后的事件，获取 JS 包文件在空闲快执行或者获取 CSS 模块在正确的时间点执行，可以说是非常强大的。

推送不能用于第三方资源的内容，通过立即发送资源，它还有效地缩短浏览器自身的资源优先级情况。在你明确的知道在做什么时，这应该会提高你的应用性能，如果不是很清晰的话，你也许会损失掉部分的性能。

<section class="" style="box-sizing: border-box; font-size: 16px; text-align: left; margin-top: 30px; margin-left: 8px; color: rgb(60, 112, 198);"><span style="display: inline-block; width: 15px; height: 15px; margin-right: 10px; background-image: url(&quot;http://mmbiz.qpic.cn/mmbiz_png/uMh5nccSicmLhOGj31Ox7URHHfI7myYCgHnPNoGLcfCtRCsGgZB6ytGYqXVfn99IqJqo77u8icvnzZic7ibiaZoLPtA/0?wx_fmt=png&quot;); background-position: center center; background-repeat: no-repeat; background-attachment: initial; background-origin: initial; background-clip: initial; background-size: 100% 100%;"></span>preload HTTP 头是什么？跟 preload 标签有什么不同？又跟 HTTP/2 服务器推送有什么不同？</section>

跟其他链接不同，preload 链接即可以放在 HTML 标签里也可以放在 HTTP 头部（<a style="color: #5baceb; word-break: break-all;">preload HTTP 头</a>），每种情况下，都会直接使浏览器加载资源并缓存在内存里，表明页面有很高的可能性用这些资源并且不想等待 preload 扫描器或者解析器去发现它。

当金融时报在它们的网站使用 preload HTTP 头时，他们节约了**大约 <a style="color: #5baceb; word-break: break-all;">1s</a> 的显示片头图片时间**。

<a style="color: #5baceb; word-break: break-all;">![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)</a>

下面的：使用 preload，上面：使用 preload。在 3G 网络下的 Moto G4 测试。

*   原来：<a style="color: #5baceb; word-break: break-all;">https://www.webpagetest.org/result/170319_Z2_GFR/</a>

*   之后：<a style="color: #5baceb; word-break: break-all;">https://www.webpagetest.org/result/170319_R8_G4Q/</a>

<a style="color: #5baceb; word-break: break-all;">你可以使用两种形式的</a> preload，但应当知道很重要的一点：根据规范，许多服务器当它们遇到 preload HTTP 头会发起 HTTP/2 推送，HTTP/2 推送的性能影响不同于普通的预加载，所以你要确保没有发起不必要的推送。

你可以使用 preload 标签来代替 preload 头以避免不必要的推送，或者在你的 HTTP 头上加一个 “nopush” 属性。

<section class="" style="box-sizing: border-box; font-size: 16px; text-align: left; margin-top: 30px; margin-left: 8px; color: rgb(60, 112, 198);"><span style="display: inline-block; width: 15px; height: 15px; margin-right: 10px; background-image: url(&quot;http://mmbiz.qpic.cn/mmbiz_png/uMh5nccSicmLhOGj31Ox7URHHfI7myYCgHnPNoGLcfCtRCsGgZB6ytGYqXVfn99IqJqo77u8icvnzZic7ibiaZoLPtA/0?wx_fmt=png&quot;); background-position: center center; background-repeat: no-repeat; background-attachment: initial; background-origin: initial; background-clip: initial; background-size: 100% 100%;"></span>我怎样检测 link rel=preload 的支持情况呢？</section>

用下面的代码段可以检测`<link rel=”preload”>`是否被支持：

```
const preloadSupported = () => {
      const link = document.createElement('link');
      const relList = link.relList;
      if (!relList || !relList.supports)
        return false;
      return relList.supports('preload');
    };
```

FilamentGroup 也有一个 <a style="color: #5baceb; word-break: break-all;">preload</a> 检测器 ，作为他们的异步 CSS 加载库 <a style="color: #5baceb; word-break: break-all;">loadCSS</a> 的一部分。

<section class="" style="box-sizing: border-box; font-size: 16px; text-align: left; margin-top: 30px; margin-left: 8px; color: rgb(60, 112, 198);"><span style="display: inline-block; width: 15px; height: 15px; margin-right: 10px; background-image: url(&quot;http://mmbiz.qpic.cn/mmbiz_png/uMh5nccSicmLhOGj31Ox7URHHfI7myYCgHnPNoGLcfCtRCsGgZB6ytGYqXVfn99IqJqo77u8icvnzZic7ibiaZoLPtA/0?wx_fmt=png&quot;); background-position: center center; background-repeat: no-repeat; background-attachment: initial; background-origin: initial; background-clip: initial; background-size: 100% 100%;"></span>你可以让 preload的 CSS 样式表立即生效吗？</section>

当然，preload 支持基于异步加载的标记，使用 `<link rel=”preload”>` 的样式表使用 `onload` 事件立即应用到文档：

```
<link rel="preload" href="style.css" onload="this.rel=stylesheet">
```

更多相关的例子，看一下 Yoav Weiss 很棒的<a style="color: #5baceb; word-break: break-all;">使用实例</a>。

<section class="" style="box-sizing: border-box; font-size: 16px; text-align: left; margin-top: 30px; margin-left: 8px; color: rgb(60, 112, 198);"><span style="display: inline-block; width: 15px; height: 15px; margin-right: 10px; background-image: url(&quot;http://mmbiz.qpic.cn/mmbiz_png/uMh5nccSicmLhOGj31Ox7URHHfI7myYCgHnPNoGLcfCtRCsGgZB6ytGYqXVfn99IqJqo77u8icvnzZic7ibiaZoLPtA/0?wx_fmt=png&quot;); background-position: center center; background-repeat: no-repeat; background-attachment: initial; background-origin: initial; background-clip: initial; background-size: 100% 100%;"></span>preload 还有哪些更广泛的应用？</section>

**根据 HTTPArchive，<a style="color: #5baceb; word-break: break-all;">很多网站</a>应用 `<link rel=”preload”>` 来加载<a style="color: #5baceb; word-break: break-all;">字体</a>，包括 Teen Vogue 和以上提到的其他网站：**

<a style="color: #5baceb; word-break: break-all;">![](http://mmbiz.qpic.cn/mmbiz_png/uMh5nccSicmLhOGj31Ox7URHHfI7myYCgvcQYhIpe3CGKjEGs6Ak56svSZUqL0ialpLcxzkibpIQ1hmcDibLu1Dib3g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)</a>

**<a style="color: #5baceb; word-break: break-all;">其他</a>一些网站，比如 LifeHacker 和 JCPenny 用 FilamentGroup 的 <a style="color: #5baceb; word-break: break-all;">loadCSS</a> 来异步加载 CSS:**

<a style="color: #5baceb; word-break: break-all;">![](http://mmbiz.qpic.cn/mmbiz_png/uMh5nccSicmLhOGj31Ox7URHHfI7myYCg8fJHEYFNqDWvicQZCUbplEXNa5Jg2GI1jlyOBQNynLHGoCAicjEwrEjw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)</a>

**有越来越多的渐进式 Web 应用程序（比如 Twitter.com 移动端， Flipkart 和 Housing）使用它来加载当前链接需要的脚本：**

<a style="color: #5baceb; word-break: break-all;">![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)</a>

基本的观点是要保持高粒度而不是单片，所以任何应用都可以按需加载依赖或者预加载资源并放在缓存中。

<section class="" style="box-sizing: border-box; font-size: 16px; text-align: left; margin-top: 30px; margin-left: 8px; color: rgb(60, 112, 198);"><span style="display: inline-block; width: 15px; height: 15px; margin-right: 10px; background-image: url(&quot;http://mmbiz.qpic.cn/mmbiz_png/uMh5nccSicmLhOGj31Ox7URHHfI7myYCgHnPNoGLcfCtRCsGgZB6ytGYqXVfn99IqJqo77u8icvnzZic7ibiaZoLPtA/0?wx_fmt=png&quot;); background-position: center center; background-repeat: no-repeat; background-attachment: initial; background-origin: initial; background-clip: initial; background-size: 100% 100%;"></span>当前浏览器对 preload 和 prefetch 的支持度？</section>

根据 CanIUse 在 <a style="color: #5baceb; word-break: break-all;">Safari Tech Preview</a>的调查看，`<link rel="preload">` 大约有 <a style="color: #5baceb; word-break: break-all;">50%</a> 的支持度，`<link rel="prefetch">` 大约有 <a style="color: #5baceb; word-break: break-all;">70%</a> 的支持度。`<link rel="preload">` is available to <a style="color: #5baceb; word-break: break-all;">~50%</a> of the global population according to CanIUse and is implemented in the <a style="color: #5baceb; word-break: break-all;">Safari Tech Preview</a>. `<link rel="prefetch">` is available to <a style="color: #5baceb; word-break: break-all;">71%</a> of global users.
