# 使用RUM度量和优化单页应用程序的性能

来源：[LinkedIn Engineering Blog](https://engineering.linkedin.com/blog/2017/02/measuring-and-optimizing-performance-of-single-page-applications)

原作者：[Sreedhar Veeravalli](https://www.linkedin.com/in/sreedharbabu)

译制：[Tingrui Li](https://www.linkedin.com/in/tingruili/)

本文主要介绍LinkedIn测量和优化单页应用程序（SPA）渲染页面效率的方案。

## RUM - 真实用户监控

RUM 是指Real User Monitoring，是指采用真实用户数据，而不是实验/测试环境，来度量性能表现，这也是领英度量网站速度的主要方法。

**页面加载时间** 是一个关键指标，用以度量从用户的角度看到 **页面从开始加载到加载完成所花费的时间** ，对于传统web应用程序，大多数页面都是服务器渲染的，用window.onload()事件作为时间节点来测量页面加载时间是一个比较好的方法。然而，在使用这个方法测量SPA的加载时间时会遇到一些障碍。

## SPA - 单页应用程序

单页应用程序（SPA）是使用Javascrpt MVC框架（例如Ember等）构建的web应用程序，来提供一个丰富的，类似app的使用体验。这些程序的HTML通常是在客户端浏览器而非服务器端渲染。这些程序中的页面/URL能以两种方式访问：

- 应用启动：在浏览器中输入URL或者点击链接时触发。"应用启动"模式通常较慢，因为JS/CSS需要被下载并且运行，才能开始渲染页面。
- 子序列：当程序已经被加载，并且在程序内部跳转时发生。这时访问速度比较快，因为应用程序的代码已经被下载并执行过，我们只需要获取新的数据并且渲染出来。

领英的web应用程序从传统服务器端渲染迁移到SPA后，使用window.onload()事件作为时间节点来度量性能会遇到许多问题。

当页面以"应用启动"模式被访问时，在用户看到有意义的内容之前window.onload()会过早触发，如下图所示。window.onload()事件在页面还没开始渲染（用户看不到页面上任何内容）的时候就已经触发了。

![时间线](https://content.linkedin.com/content/dam/engineering/site-assets/images/blog/posts/2017/02/SRA1.jpg)

当页面以"子序列"模式被访问时，windows.onload()根本不会触发，因为没有新的HTML文档被下载。所以不能依赖window.onload()来进行性能跟踪。

![](https://content.linkedin.com/content/dam/engineering/site-assets/images/blog/posts/2017/02/SRA2.jpg)

## 针对SPA的RUM

我们需要检测"子序列"访问的开始，检测页面何时开始渲染以测量 **页面加载时间**。我们考虑了以下方法：

- 用[Resource Timing API](https://www.w3.org/TR/resource-timing/)来检测AJAX请求发起以标志"子序列"访问的开始。
- 用[MutationObserver](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver)检测DOM元素的变化，和用Resource Timing API来检测网络活动的终点。

这些方法都不可靠、不准确。比如，我们可能使用AJAX请求 **提前获取** 一些数据，甚至在用户跳转页面触发"子序列"访问之前。同时，一些网络活动可能并不会对用户体验造成影响。我们需要页面加载时间的计算起点足够接近用户开始看到内容的时间，而非网络活动的发起时间。

幸好，一些SPA框架比如Ember，拥有丰富的[生命周期方法](http://emberjs.com/api/classes/Ember.Router.html)，能够为我们的测量提供便利。

```javascript
// Add instrumentation for subsequent navigation start
router.on('willTransition', () => {
    rumObj.appTransitionStart();
});

// Add instrumentation for rendering is complete
router.on('didTransition', () => {
    Ember.run.scheduleOnce('afterRender', () => {
        rumObj.appRenderComplete();
   });
});
```

我们通过[navigationStart]API可以知道"应用启动"的开始时间，通过router的willTransition事件可以知道"子序列"访问的开始时间。我们听取router的didTransition事件并且向afterRender队列中加入一些东西来得知页面渲染完成。我们现在有"应用启动"模式访问和"子序列"模式访问的开始和结束时间了，所以可以从中计算出 **页面加载时间** 。

通过这种方法，我们已经将 **页面加载时间** 的自动化测量方案应用在领英的许多单页应用程序中，它们大多基于 [Ember](http://emberjs.com/)、[AngularJS](https://angularjs.org/)和[Marionette](http://marionettejs.com/)框架。

## 寻找优化机会

如果想要调试性能问题，需要分析页面加载时间以寻找瓶颈所在。相对于传统Web应用的HTML和资源下载时间，单页应用程序有相当一部分时间是花在浏览器在下载完HTML/JS/CSS后的JS执行时间。为了找到这些时间的关键时间点和关键阶段，我们选择使用[User Timing API](https://www.w3.org/TR/user-timing/)来添加进行度量。在有了这些信息后，就可以构建下面的瀑布图，帮助我们可视化整个流程以寻找瓶颈所在。 ![](https://content.linkedin.com/content/dam/engineering/site-assets/images/blog/posts/2017/02/SRA3.jpg)

下面提供几个根据RUM数据进行优化的例子。

## 懒渲染

分析这些信息时，我们发现有大概30%的页面加载时间消耗在"渲染"阶段。渲染阶段是当数据可用以后在页面上构建DOM元素所花的时间。我们也注意到浏览器的主线程没有进行绘制，直到页面上所有组件DOM元素都被创建。

![](https://content.linkedin.com/content/dam/engineering/site-assets/images/blog/posts/2017/02/SRA4.jpg)

如果我们能早一些启动浏览器主线程进行页面绘制，比如viewport（浏览器当前用户可见范围）内的DOM已经创建时，可以得到一个更加快速的反馈，从用户体验角度来说速度就快多了。

在更多的层面上，主要的性能问题是页面上所有组件拥有同样的优先级并且总是同时被渲染/绘制。所以我们需要有一个方案来划分优先级，基于哪些元素需要何时被渲染。如下图：

![](https://content.linkedin.com/content/dam/engineering/site-assets/images/blog/posts/2017/02/SRA5.jpg)

第一类：蓝色，总是需要被渲染，无关viewport高度。

第二类：绿色，优先级低一些，可以在下一次动画帧中逐步渲染，根据两个条件：

- 如果当前viewport比典型的viewport更高，这些组件会被渲染；
- 如果需要流畅的滚动体验（比如通过滚动条或鼠标滚轮），这些组件会被渲染，就算在viewport之外。

第三类：红色，这些组件一开始不在viewport之内就不会被渲染。当滚动至它们进入viewport时进行渲染。

采取这个方案，我们可以拥有一个更快的[首次有效绘制](https://developers.google.com/web/tools/lighthouse/audits/first-meaningful-paint)。我们对一些页面采用了这个方案，发现在渲染阶段的时间有50%的性能提升。

![](https://content.linkedin.com/content/dam/engineering/site-assets/images/blog/posts/2017/02/SRA6.jpg)

## 懒数据获取

当我们优化了渲染阶段后，我们发现20%的页面加载时间被消耗在"过渡"阶段。这个阶段是我们等待数据到达，格式化数据，然后将其推送至Ember数据储存。这里的时间消耗很大程度上取决于有多少数据需要处理。如果我们能够对[首次有效绘制](https://developers.google.com/web/tools/lighthouse/audits/first-meaningful-paint)所 **不需要的数据** 进行懒获取，也能够减少需要处理的数据量和得到一个更快的首次有效绘制。

所以我们将数据获取过程分成两个层面：首次有效绘制所需数据和可以在之后再进行获取的数据。我们执行了这些优化并发现过渡阶段的效率提升了40%。

## 总结

通过由RUM数据实现和验证的优化流程，我们总结出一些最佳实践和经验教训：

- 推迟首次有效绘制不需要的工作。这条原则在很多情况下能带来很好的结果，有许多其他的优化方法，比如懒加载首次有效绘制所需的应用代码（JS/CSS），也在此列。
- 除了从开发环境和人工预设环境中收集性能数据，彻底分析RUM数据很有帮助。有很多种设备访问你的站点，他们处于不同的网络环境，用户也截然不同。想在一个人工设置的环境里面模拟所有这些是不容易的。基于现实世界数据进行优化通常比针对人工设置环境收集来的数据进行优化来的更有效。
- 总是做一些A/B实验，以准确的度量特定优化所带来的收益。长期保持做这些实验，测量优化的效率，因为应用性能的瓶颈会随着时间变化。
- 为了快速的遍历新的想法，我们应该在人工性能测试环境中验证并且改进它们。构建一个独立的性能测试环境，可以让我们在有了新的优化思路后快速测试效果，以筛选有效的办法。
