---
layout: post
title: 'ZeroMQ: 其实我是一个演员'
date: 2012-01-16 15:59:20.000000000 +08:00
categories:
- 技术
tags:
- tech
- zeromq
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  dsq_thread_id: '1532653361'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
![](assets/logo.gif)

最早听说zeromq是11年年初，心想这可能又是个类似于ActiveMQ, RabbitMQ的东东，就没太在意，转身做其他事情去了。而这一转就是一年，直到一周前的一个偶然机会，我点进了它的主页仔细看了看才发现------被这货骗了！这货根本不是MQ啊！

> The Intelligent Transport Layer

这是[http://www.zeromq.org/][0]里的副标题。直译过来叫作"智能传输层"。在粗读了这个[很长很长但是行文很幽默的guide][1]之后，我觉得这个副标题才是这货真正的名字。它的大名ZeroMQ (MQ------只是**有些**使用它的程序的行为表现有那么一点像是个Message Queue罢了)实在是太牵强、太误导人了。这就好比给功能强大的智能Android系统起个叫"多媒体机"的名字一样的荒诞。好吧，也许像我这样望文生义的人也许并不多，但我不得不说这不是一个好名字。

事实上，抛开对这个名字的调侃，我对这个产品剩下的只剩下尊敬了。看看它到底是什么吧：

1. 它只是一个lib。不是什么可以用个类似于start的命令启动起来的service或者daemon程序。
2. 它是个协议。用于node之间消息的传送。这个**node**可以是线程、进程或者物理box。**之间**可以是他们任意两个之间。
3. 它不支持持久化。
4. 它的实现不包含数据的序列、反序列化。
5. 它实现了支持[Pragmatic General Multicast][2] 的广播。
6. 它高度概括并实现了三种通讯模式：Req-Rep; Pub-Sub; Pipe; [任何分布式，并行的需求，都可以用这三种模型组合起来解决问题。][3]
7. 它不是什么MQ，它其实是个演员

强烈建议精读它的[guide][1]。虽然我还没有做到这一点，但正打算这么做，后面会记些笔记贴出来。

[zeromq社区站点][4]信息量巨大，由此也可看出这是一个比较成熟的产品和社区，绝对值得加入学习研究。

此事教训深刻------对待任何事、人都不能只看表象。带翅膀的不一定是天使，有可能是鸟人；大胡子的不一定是IT大牛，也可能是宋山木。

[0]: http://www.zeromq.org/
[1]: http://zguide.zeromq.org/page:all
[2]: http://en.wikipedia.org/wiki/Pragmatic_General_Multicast
[3]: http://blog.codingnow.com/2011/02/zeromq_message_patterns.html
[4]: http://www.zeromq.org/community