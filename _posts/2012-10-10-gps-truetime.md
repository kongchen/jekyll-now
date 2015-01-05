---
layout: post
title: GPS时间校准原理
date: 2012-10-10 17:49:06.000000000 +08:00
categories:
- 技术
tags:
- GPS
- truetime
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  dsq_thread_id: '1532654892'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
Google最近public了自己的一个叫做Spanner的所谓全球分布式数据库，[这篇文章][0]对此作了详细的介绍。e文过关的筒子可以去看google research上的原版paper。  
其中比较亮的是Google自己搞了一个被称为TrueTime的API作为这套数据库的基础时间组件，这个TrueTime API能够将不同数据中心的时间偏差缩短在10ms内。它的实现靠的是GFS和原子钟。之所以要用两种技术来处理，是因为导致这两个技术的失败的原因是不同的。GPS会有一个天线，电波干扰会导致其失灵。原子钟很稳定。当GPS失灵的时候，原子钟仍然能保证在相当长的时间内，不会出现偏差。

对这套实现的原理很好奇，虽然Spanner的paper并没有详细介绍如何利用GPS和原子时钟来得到精确时间，但搜了一下发现这其实是一种比较成熟的东西了，比如在对时间要求非常高的股票交易系统里就有采用。学习了一下，写篇blog做个总结。

根据[wikipedia的解释][1]，要利用GPS得到精确标准时间至少需要4颗GPS卫星。  
为什么是4颗呢？我们看下面这个图：  
![](assets/gps.jpg)  
图里的球是我们的地球，假设赤道平面为xy平面，地心为原点，北极点为z轴方向。那么我们就有了一个xyz的空间，A, B, C, D是四颗GPS卫星。我们的接收器在地表，那么ABCDE以及接收器都会有各自的坐标。  
已知条件是ABCD都知道自己的坐标，因为它们都是地球同步卫星，事实上他们也会在导航电文中广播自己的轨道坐标(xi, yi, zi)。  
另一个也会被广播的信息非常重要，它就是每一个GPS卫星上的时间ti。我们可以把这个时间视为理想标准时间，这可以让我们的工作大幅度简化。（虽然它们也跟理论上的理想标准时间有误差，但是前人做了很多的工作把这个误差变得微乎其微了）

那么，继续看图。假设在![](assets/b131e78b2517a5a64f6e90c0935f2ea5.png)时，接收器收到了卫星i发来的信息（包含\[xi, yi, zi, ti\]）。而真正的接收时间将是![\, t_\text{r} + b](assets/e85dd51682137d11c6f7053c90c368aa.png), 其中![\, b ](assets/383bf8ddc91e5cd69aee64fa6cff3d1f.png)是接收器的时钟偏移。所以，真正实际的传输时间就是![\, t_\text{r} + b - t_i](assets/76f7e7aa2b1197d43463a5a2e88e49fe.png) ，传输距离就是![\, \left( t_\text{r} + b - t_i \right) c](assets/6928f28fbe7c0a42b9cc3b22805ab5b0.png)。

再根据上图的三维空间，传输距离也就是卫星和接收器之间的距离，因此就有了：

![(x-x_i)^2 + (y-y_i)^2 + (z-z_i)^2 = \bigl([ t_\text{r} + b - t_i]c\bigr)^2, \; i=1,2,\dots,n](assets/6abc059d50a257b6c73cdba12956480b.png)

式子里面有\[x,y,z,b\] 一共4个未知数，所以我们只需要n=4，有4颗卫星即可算出这4个数。

一旦拿到了接收器的时钟偏移，即便在没有GPS信号的时候，我们也可以按照这个偏移精准地让本地时间向前推移。例如：本地时钟是1Ghz，偏移![\, b ](assets/383bf8ddc91e5cd69aee64fa6cff3d1f.png)按上面的方法算出来是-100。那么本来时钟每摆动1G下我们让时间走1s，现在则只需摆动(1G-100)下就走1s。由此便能在再次接收到GPS信号时间之前，最大程度地保证时间正确。

[0]: http://qing.weibo.com/2294942122/88ca09aa3300221n.html "Google Spanner原理- 全球级的分布式数据库"
[1]: http://en.wikipedia.org/wiki/Global_Positioning_System