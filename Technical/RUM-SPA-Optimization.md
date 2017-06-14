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

当页面以“应用启动”模式被访问时，在用户看到有意义的内容之前window.onload()会过早触发，如下图所示。window.onload()事件在页面还没开始渲染（用户看不到页面上任何内容）的时候就已经触发了。

![时间线](https://content.linkedin.com/content/dam/engineering/site-assets/images/blog/posts/2017/02/SRA1.jpg)

当页面以“子序列”模式被访问时，windows.onload()根本不会触发，因为没有新的HTML文档被下载。所以不能依赖window.onload()来进行性能跟踪。

![](https://content.linkedin.com/content/dam/engineering/site-assets/images/blog/posts/2017/02/SRA2.jpg)

## 针对SPA的RUM
我们需要检测“子序列”访问的开始，检测页面何时开始渲染以测量 **页面加载时间**。我们考虑了以下方法：
- 用[Resource Timing API](https://www.w3.org/TR/resource-timing/)来检测AJAX请求发起以标志“子序列”访问的开始。
- 用[MutationObserver](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver)检测DOM元素的变化，和用Resource Timing API来检测网络活动的终点。

这些方法都不可靠、不准确。比如，我们可能使用AJAX请求 **提前获取** 一些数据，甚至在用户跳转页面触发“子序列”访问之前。同时，一些网络活动可能并不会对用户体验造成影响。我们需要页面加载时间的计算起点足够接近用户开始看到内容的时间，而非网络活动的发起时间。

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

我们通过[navigationStart]API可以知道“应用启动”的开始时间，通过router的willTransition事件可以知道“子序列”访问的开始时间。我们听取router的didTransition事件并且向afterRender队列中加入一些东西来得知页面渲染完成。我们现在有“应用启动”模式访问和“子序列”模式访问的开始和结束时间了，所以可以从中计算出 **页面加载时间** 。

通过这种方法，我们已经将 **页面加载时间** 的自动化测量方案应用在领英的许多单页应用程序中，它们大多基于 [Ember](http://emberjs.com/)、[AngularJS](https://angularjs.org/)和[Marionette](http://marionettejs.com/)框架。
