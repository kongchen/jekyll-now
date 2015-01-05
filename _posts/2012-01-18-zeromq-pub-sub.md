---
layout: post
title: ZeroMQ的模式-Publish-Subscribe
date: 2012-01-18 16:38:54.000000000 +08:00
categories:
- 技术
tags:
- Pattern
- pub-sub
- zeromq
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  dsq_thread_id: '1533027131'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
**Publish-subscribe Pattern**：发布订阅模式。

现实中，并不是所有请求都期待答复，而不期待答复，自然就没有了状态。所以相对于REQ-REP，PUB-SUB模式容易理解也简单得多。广播听过吧?收音机用过吧?就这个意思。

相应地，该模式下的socket也就两种：ZMQ\_PUB & ZMQ\_SUB。 分别对应电台和收音机。

# ZMQ\_PUB

ZMQ\_PUB主要用来让消息发布者用来散发消息的。所有连接上的peer都能收到由它散发的消息。 [zmq\_recv(3)][0] 这个API是不能用在这个socket上的，原因显而易见。而zmq\_send作用在该socket上时是永远不会阻塞的，如果订阅者异常，发出的消息则会被丢弃。
Summary of ZMQ\_PUB characteristics

Compatible peer sockets
_ZMQ\_SUB_

Direction
Unidirectional

Send/receive pattern
Send only

Incoming routing strategy
N/A

Outgoing routing strategy
Fan out

ZMQ\_HWM option action
Drop

# ZMQ\_SUB

很明显，订阅者通过这个socket来接受发布者发布的消息。需要注意的是，在使用该socket时，必须显式地调用 zmq\_setsockopt ，设置ZMQ\_SUBSCRIBE和filter。如果不设置的话，是收不到任何消息的。
Summary of ZMQ\_SUB characteristics

Compatible peer sockets
_ZMQ\_PUB_

Direction
Unidirectional

Send/receive pattern
Receive only

Incoming routing strategy
Fair-queued

Outgoing routing strategy
N/A

ZMQ\_HWM option action
Drop

# 总结

PUB-SUB模式一般处理的都不是系统的关键数据。发布者不关注订阅者是否收到发布的消息，订阅者也不知道自己是否收到了发布者发出的所有消息。你也不知道订阅者何时开始收到消息。因此逻辑上，它都不是可靠的。

事实上，即便你先启动订阅者，再启动发布者。订阅者也不一定能收到所有的消息。原因在于：发布者已启动就开始撒布消息，而这时订阅者可能还没有完成连接。如果一定需要保证，则需要做两者的同步。最傻的方法就是让发布者启动之后sleep一会儿再开始发消息，不过这种方式就跟听起来一样不靠谱。

一个订阅者可以订阅多个发布者。同时订阅者通过filter来过滤自己需要的消息，需要注意的时，filter是在订阅端起作用的。也就是说所有消息是会到达所有订阅者处，订阅者根据filter丢掉自己不需要的消息。

[0]: http://api.zeromq.org/2-1:zmq_recv