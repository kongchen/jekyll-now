---
layout: post
title: 怎么才能把CAP理论说通
date: 2014-03-29 10:48:12.000000000 +08:00
categories:
- 技术
tags:
- CAP
status: publish
type: post
published: true
meta:
  _wpcom_is_markdown: '1'
  _edit_last: '1'
  _wpas_done_all: '1'
  dsq_thread_id: '2968783000'
  _wp_old_slug: '%e6%80%8e%e4%b9%88%e6%89%8d%e8%83%bd%e6%8a%8acap%e7%90%86%e8%ae%ba%e8%af%b4%e9%80%9a'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
相信但凡知道点分布式的人都对CAP理论耳熟能详，可是能真正说清楚的却没几个。这理论咋一听觉得很好理解，但真正要解释给别人时却总是难以自圆其说。跟同事讨论感觉也是大家各有各的理解，像是一千个人眼中有一千个哈姆雷特,原因何在呢？

最近看了[《NoSQL Distilled》][0] 这本书，算是知道了病根儿。而且书中也提到了"理解"和"争论"这两件事:

> The basic statement of the CAP theorem is that, given the three properties of  
> Consistency, Availability, and Partition tolerance, you can only get two.  
> Obviously this depends very much on how you define these three properties, and differing opinions have led to several debates on what the real consequences of the CAP theorem are.
> 

那我们就来看看CAP的字面意思：

* **C** 一致性（Consistency) 
* **A** 可用性（Availability）
* **P** 分区容忍度（Partition tolerance）  
CAP理论就是在一个分布式系统里，最多只能满足C、A、P中的两个。

C容易理解，所有节点上的数据都一样。  
P是说一旦集群因为内部通信故障发生分裂，集群还能正常运转。

关键是这个A。

咋一看Availability这个简直太好理解了，而且我们一点也不陌生。哪个系统设计里少得了它？通常指的是系统能正常工作，系统架构所追求的HA即保证系统高可用，不宕机。企业级的4个9,5个9的，说的就是这个A。

但CAP理论里这个**A**却不是这个意思，[《NoSQL Distilled》][0] 里对这个问题专门做了澄清：

> By the usual definition of "available," this would mean alack of availability, but this is where CAP's special usage of "availability" gets confusing. CAP defines "availability" to mean "every request received by a non-failing node in the system must result in a response"\[Lynch and Gilbert\]. So a failed, unresponsive node doesn't infer alack of CAP availability.
> 

说白了就是，CAP里的A指的是，只要是活着的节点能返回响应，那么就认为它是availability的。

用汽车来做个比喻：  
汽车一切良好，故障灯一个都不亮，能开能刹，这是我们理解的传统的Available。只要是出了任何问题，例如车胎爆了，发动机坏了，或者是没油开不动了，这都已经是**不可用**了。

CAP理论里，车不能开不要紧，只要是这车车门还能打开，那就是Available的：车胎破了能从胎压监测里看到，发动机坏了打火打不着，没油了油表灯会亮......这都是车返回的response，只要是有response，那么就是"**可用的**"。

理解了A的真正含义以后再来看CAP。

* 只满足CA的系统  

> 根据P的定义： _一旦集群因为内部通信故障发生分裂，集群还能正常运转。_ 舍弃P意味着一旦集群发生分裂，整个集群都将无法运转。这是符合A的，因为这是的节点是fail的，不需要给任何响应。  
> 单机服务器显然是CA的，只要结点活着，那自然是C+A。而结点一旦挂了，整个集群也就没了，符合舍弃P的选择。
> 

* 只满足CP的系统  

> 一旦集群内部因为内部通信故障发生分裂（假设分裂成两部分），为了满足P，集群需要提供服务。而为了满足C，只能保留其中一部分提供服务，让另一部分整体退役。
> 

* 只满足AP的系统  

> 整个最容易理解，一旦集群内部因为内部通信故障发生分裂（假设分裂成两部分），为了满足P，而又不需要满足C，那么可以让分裂出来的两部分都提供服务，因为暂时数据不一致没关系。
> 

总算说圆了。

## 后记

后来发现其实CAP理论的提出者之一在[2012年写文章][1] 解释了为什么3选2这么难说通，原因就是这个3选2是被误读了-\_-!!  
意思是说CAP理论也是与时俱进的，在当前的IT环境下，压根不用去讨论什么CA怎么解释，因为P是一定的。可以用下面这个图来表达现在CAP:  
![cap-teaser](assets/cap-teaser1-300x142.png)

[0]: http://www.amazon.com/NoSQL-Distilled-Emerging-Polyglot-Persistence/dp/0321826620
[1]: http://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed