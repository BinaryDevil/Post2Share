# 外文干货译制节选之 - 使用Brotli压缩加快网站响应速度

来源：[LinkedIn Engineering Blog](https://engineering.linkedin.com/blog/2017/05/boosting-site-speed-using-brotli-compression)

原作者：[Ruobing Li](https://www.linkedin.com/in/ruobingli)

译制：[Tingrui Li](https://www.linkedin.com/in/tingruili/)

网站响应速度一直是领英在工程中的优先考虑项，因为更快的响应速度直接带来更高的用户体验。对于提高网站响应速度有许多种方法，显然其中一种就是减少需要下载到浏览器中的字节数。为此，你不只是要优化你的JS和CSS的文件大小，更需要一个卓越的压缩算法来减少传输的字节量。

如今最主流的压缩算法是gzip。从2017年初开始，我们开始偏好将Brotli而非gzip用于大部分领英的网页。如果你的浏览器支持Brotli（下方有相关详情），我们会发送Brotli压缩过的静态内容（Javascript和CSS）。我们的实验证明，Brotli能带来显著的响应速度提升，特别是在像印度这样的市场，因为缺少带宽人们较难获得良好的体验。

## 什么是Brotli
Brotli是一个新的，开源的压缩算法，由两位Google工程师开发，于2015年发布。它是设计来压缩文本内容，并且相较于gzip压缩后容量还要再小20%-30%。他的编码速度广泛地慢于gzip（取决于压缩质量设定），但解码速度跟gzip不相上下。

### 浏览器兼容性
时至今日，以下主流浏览器支持Brotli：

* Chrome：Version 49及以上
* Microsoft Edge：将从下一个版本（15）开始支持
* Firefox：Version 44及以上
* Opera：Version 36及以上

Safari目前没有对Brotli公开表态。没有任何一个IE版本支持Brotli，但大多数Windows用户现在都来自Edge，Chrome或者Firefox。对于移动web，浏览器更碎片化，对其的支持要更糟糕。

![browser support](https://content.linkedin.com/content/dam/engineering/site-assets/images/blog/posts/2017/05/brotli1.jpg)

上面的图表示Brotli支持情况。全局实施率约为56.75%。我们也针对跟踪数据研究了来自三个主要国家的领英用户。
![LinkedIn users](https://content.linkedin.com/content/dam/engineering/site-assets/images/blog/posts/2017/05/brotli3.jpg)

桌面用户主要来自Chrome和Firefox，移动端用户主要是Safari（中国和美国）和Chrome（印度）。大多数印度的会员使用Chrome促成了Brotli拥有更高的实施率。

支持Brotli的浏览器自动在HTTP请求里的`Accept-Encoding`头中包含br，这告诉服务器这个user agent能够解压Brotli的response。

## CDN支持
时至今日，CDN对Brotli的支持较不广泛。领英使用5个不同的CDN供应商。目前为止，这五家都没有公开宣布支持Brotli。如果你想结合这些CDN使用，有非常大的可能性不会直接生效。他们都有内置的gzip用例假设。

好消息是，因为设计和配置得当，我们成功地使5家CDN供应商承载和缓存Brotli静态内容。

## Brotli vs Zopfli vs gzip
根据我们的评分比较，我们发现Brotli -11较gzip -6压缩大小还要小30%。并且它的解压速度稍微比gzip和Zopfli快些。
![compression comparison](https://content.linkedin.com/content/dam/engineering/site-assets/images/blog/posts/2017/05/brotli4.jpg)

Algorithm	| Quality	| Compression Time (ms)	| Decompression Time (ms)
--- | --- | --- | ---
gzip |	6	| 169 |	35
gzip |	9	| 284 |	27
zopfli |	15 |	37,847 |	32
zopfli |	100 |	194,460 |	38
zopfli |	1000 |	1,855,480 |	29
brotli |	4 |	109 |	24
brotli |	5 |	193 |	20
brotli |	9 |	517 |	23
brotli |	11 |	11,913 |	22

## 实现
### 静态内容服务器
领英使用专用中央服务器来作为应用静态内容的CDN源，应用web服务器无需负责此事。每项静态内容以它的内容的MD5 hash唯一标记。一个典型的领英静态内容是：

`https://static.licdn.com/sc/h/<hashes>`

`static.licdn.com`所有领英的CDN供应商的一个雨伞域名。如果缓存未命中，CDN从我们的静态内容源服务器获取目标，否则直接返回缓存中的目标。

当一个web应用部署到生产环境中，应用程序的静态内容被上传并存储在我们的呢Espresso数据库。计算出MD5值用来作键以索引文件。raw文件和Brotli压缩文件有不同的hash值和URL，内容也不相同。静态内容服务器识别带有.br后缀的文件，并储存压缩后的二进制内容。当一个GET请求到达时，它将回传被压缩的二进制内容并且加入一个`Content-Encoding: br`在response头里。

### Build流水线
既然我们在讨论静态内容，事先采取最佳质量设定（-11）来压缩文件似乎更有意义。为了让build时间维持在正常水平，我们只压缩JS和CSS因为他们通常被归为“关键资产”。对于领英这样的单页应用程序（SPA）来说，这些文件的可用性显著影响到页面渲染时间线。

对于每个文件，我们使用Brotli CLI压缩后写到一个.br后缀的文件中。build流水线同时创建一个JSON文件储存文件名和MD5 hash的映射。
```JSON
{
  "hashes": {
    "assets/core.js": "7qv1l65bu06w6si5hh38nyk2z",
    "assets/core.js.br": "6buotury4cj126ie0rgrjvd"
  }
}
```
hash值之后被用于web服务器针对任何静态内容文件生成CDN URL。

### 应用web服务器
build流水线生成每个文件的两种版本：raw和Brotli压缩文件。我们在应用服务器上实现了一个特殊的内容协商机制用来决定应该提供哪一个版本。

发到www.linkedin.com的最初请求由应用web服务器处理，它回传一个基础HTML文本，其中包含启动app所需要的所有JS和CSS的URL。这些URL是由服务器端根据客户端发送请求的`Accept-Encoding`头（Header）生成的。如果header中有`br`这一项，我们会选择Brotli压缩版本，根据之前JSON中的hash值生成CDN URL；否则我们就会使用raw文件，也就是由静态内容服务器以gzip压缩过的文件。例如，针对Safari提供的静态内容URL会不同于Chrome/Firefox，因为Safari不支持Brotli解码。

## 速度提升结果度量
我们创建了一个A/B测试，对于50%的领英用户采取“尽可能使用Brotli”方案，另外50%使用原本的办法。我们比较了领英主页加载到90%所用的时间，发现US地区有2-3.6%的响应速度提升，而在印度则有6-6.5%。Brotli对于低带宽用户能带来更大的收益。对于印度的提升大于美国，移动端的提升大于桌面端。
![Site speed improvement](https://content.linkedin.com/content/dam/engineering/site-assets/images/blog/posts/2017/05/brotli6.jpg)

## 结论
对于静态内容使用Brotli，我们看到显著的速度提升，尤其是对于低带宽的场景。我们相信Brotli具有很大潜力并且鼓励所有网站采取Brotli以获得更佳的用户体验。随着新的浏览器版本发布，随着用户浏览器的升级，这个办法的收益会随着时间逐渐增加。
