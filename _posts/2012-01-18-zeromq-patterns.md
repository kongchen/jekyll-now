---
layout: post
title: ZeroMQ的模式-综述
date: 2012-01-18 15:39:06.000000000 +08:00
categories:
- 技术
tags:
- Pattern
- zeromq
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  dsq_thread_id: '1536841794'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
通过对[Guide][0]的阅读，可以发现ZeroMQ对这个世界中消息传输的模式进行了很好的抽象。为了描述模式，0mq定义了不同的socket。 0mq socket是0mq世界的东西，跟传统世界的socket是不一样的。

我们知道，传统的socket其实就是访问下面两种(TCP & UDP)对象的**同步**的接口：

1. 面向连接的可靠字节流(SOCK\_STREAM)
2. 无连接的不可靠的数据报文(SOCK\_DGRAM)

所以你可以说传统socket传输的是字节流或者独立的报文。  

而0mq的socket传输的是消息(Message)。它是对**异步_消息_**_队列_(MQ)的一种抽象。官方的原话是：

> ØMQ sockets present an abstraction of an asynchronous _message queue_, with the exact queueing semantics depending on the socket type in use. 

**异步**的意思在这里指的是物理连接的创建、销毁、重连、传输对于用户来说都是透明的，这些东西都由0mq组织好了。它传输的是独立的**_消息_**。_队列_隐含的意思是万一消息无法到达对端则可能会被排队。

除了传统socket实现的**一对一**、**多对一**以及**一对多**(广播）外，0mq的socket还可以用zmq\_connect()发起连接到多个对端，并同时接受从多个用zmq\_bind()绑定了0mq-socket的对端发起的链接，从而实现**多对多**。

0mq归纳的模式有四种

1. Request-reply Pattern
2. Publish-subscribe Pattern
3. Pipeline Pattern
4. Exclusive pair Pattern

我想搞懂了这些模式，可能也就理解了zeromq的精髓和用法。只有这样才能灵活地、根据场景使用不同的模式，利用zeromq快速搭建网络拓扑。

[0]: http://zguide.zeromq.org/page:all