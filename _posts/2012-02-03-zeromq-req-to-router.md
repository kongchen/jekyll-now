---
layout: post
title: ZeroMQ试用笔记之REQ & ROUTER
date: 2012-02-03 15:28:38.000000000 +08:00
categories:
- 技术
tags:
- czmq
- Pattern
- ROUTER
- zeromq
- ZMQ_ROUTER
status: publish
type: post
published: true
meta:
  _syntaxhighlighter_encoded: '1'
  _edit_last: '1'
  dsq_thread_id: '1532652547'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
zeromq的尝鲜笔记之一。内容包含ROUTER socket的理解介绍，一个小代码片段，以及czmq中处理消息帧的api的用法。

试用说白了就是用zeromq写套小东西。期间必然会遇到问题，笔记的目的无非就是记录问题，加深理解。而实际上我在记录的过程中也不断地修正了一些起初想当然的错误的理解。虽说已极力避免，但由于都是自己一人的理解，错误在所难免，希望有兴趣看的同学帮我指出。

# 背景

目标是依赖Zeromq实现一个支持并发的server端，接收多个client的请求，略作处理后返回响应。

client端是windows上的java程序。server端则是CentOS下的cpp程序。

# 实现

若直接用REQ-REP模式的话，需要注意到：REP端必须严格遵循recv,send,recv,send....的步骤，倘若REP端在recv后需要一定时间的处理之后才能send，那么接下来的下一个REQ就得被迫等着了。所以从外部看，我们的REP的并发只有1。就跟下面这张图一样，步骤4的消息必须等到步骤3之后才能被接收。

![](assets/REQ-REP1.png)

这可能是有些场景下必须的，但不是我想要的。因为我们需要并发。说白了就是我们要在步骤2的执行期间，把步骤4甚至接下来的5、6都做了。

ROUTER 正是基于这样的目的才被引入的。[我们之前也有过介绍][0]，但那个介绍在我看来更像是个guide翻译。这里再加上自己的理解细说一下。

REP之所以要按部就班，因为它如果不按部就班，就不知道把响应发回给哪里，所以它必须要同步地，先recv再send。

我们再来看ROUTER。它之所以可以不按部就班，是因为它收到REQ的消息时，在消息头上加入来源地址，然后再交给客户端。发送时，取出消息第一帧作为目标地址，将空帧之后的帧进行发送。

举例来说，app1通过REQ发送给通过ROUTER接收的app2。若app1发送的是

\[code\]\["hello"\]\[/code\]

，经由ROUTER的处理，app2应用层得到的消息将是

\[code\]\[app1's address|empty|"hello"\]\[/code\]  
对于app2，不能只关心业务数据"hello"，还需要将app1's address缓存下来，用以响应的回复。比如，若app2要回复"world"，需要手动构造一个有三个frame的消息：  
\[code\]  
\[app1's address|empty|"world"\]  
\[/code\]  
再把这个消息交给ROUTER socket进行send，这时ROUTER会将第一帧address取出作为目标地址------也就是的REQ端------，再将空帧之后的数据发出。所以最终REQ端收到响应为：  
\[code\]\["world"\]\[/code\]  
这些步骤看起来复杂，看看C代码怎么写。[zeromq给我们提供了包装库czmq][1]，利用它我们可以很方便地做到上述的过程。  
这是代码片段。  
\[c\]  
void \*receiver = zsocket\_new(ctx, ZMQ\_ROUTER);  
zsocket\_bind(receiver, "ipc:///tmp/0");

....

//recv message  
zmsg\_t \*msg = zmsg\_recv(receiver);  
if (!msg){  
//error handle..  
return;  
}  
zframe\_t \*address = zmsg\_unwrap (msg);  
zframe\_t \*frame = zmsg\_first (msg);  
zmsg\_destroy (&msg);

//do something, it make time some times...

//make response message  
zmsg\_t \*message = zmsg\_new();  
zframe\_t\* body = zframe\_new("world",sizeof("world"));

zmsg\_push (message, body);  
zmsg\_wrap (message, address);

zmsg\_send (&message, receiver);  
zmsg\_destroy(&message);

\[/c\]  
需要说明的是czmq里的几个函数，万万不可用错。虽然有注释，但要用对这些api首先得搞清楚里面提到的first，front在message里是怎么个位置，为此我画了个图：  
![](assets/REQ-REP2.png)

\[c\]  
// Pop frame off front of message, caller now owns frame  
// If next frame is empty, pops and destroys that empty frame.  
zframe\_t \*  
zmsg\_unwrap (zmsg\_t \*self);  
\[/c\]  
\[c\]  
// Return first frame in message, or null  
zframe\_t \*  
zmsg\_first (zmsg\_t \*self);  
\[/c\]  
\[c\]  
// Push frame to front of message, before first frame  
void  
zmsg\_push (zmsg\_t \*self, zframe\_t \*frame);  
\[/c\]  
\[c\]  
// Push frame to front of message, before first frame  
// Pushes an empty frame in front of frame  
void  
zmsg\_wrap (zmsg\_t \*self, zframe\_t \*frame);  
\[/c\]  
所以回头看代码片段，我们在构造响应消息时先push了一个填充着"world"的帧。再zmsg\_wrap了一个填充着address的帧。  
zmsg\_wrap做了两个事情：1）在"world"之前放上address的帧； 2）在这个帧front放个空帧。

[0]: /2012/01/zeromq-pattern-requset-reply "ZeroMQ的模式-Requset-Reply"
[1]: http://czmq.zeromq.org/