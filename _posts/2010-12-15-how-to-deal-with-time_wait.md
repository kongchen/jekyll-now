---
layout: post
title: How to deal with TIME_WAIT?
date: 2010-12-15 23:07:02.000000000 +08:00
categories:
- 技术
tags:
- TCP
- TIME_WAIT
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  _wp_old_slug: ''
  _syntaxhighlighter_encoded: '1'
  dsq_thread_id: '1532938090'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
TIME\_WAIT是个很老但又很有意思的话题。

为什么说它老呢，因为比我年纪还大的[RFC793][0]就已经对它进行了定义：

> TIME-WAIT - represents waiting for enough time to pass to be sure the remote TCP received the acknowledgment of its connection termination request.

为什么说它有意思呢？因为它给我们带来了麻烦，但是我们又离不开它。

## TIME\_WAIT导致的问题

说它有意思是它确实对应用造成了显而易见的影响，并且也有很多人试图消灭TIME\_WAIT。比较常见的场景想必很多人都遇到过：client端与server端连接，处理完数据后。client端主动关闭了连接，此时在client端_netstat -an_会发现之前ESTABLISHED的连接会很快(通常情况下)变成TIME\_WAIT，隔段时间后（2MSL，Maximum Segment Lifetime）才从netstat的list里消失。

好像这也没什么问题，但如果此时client又想用同样等端口向server发起请求，那bind就会出问题。

同样地，如果server端主动关闭的话，那么在高并发情况下(e.g. 大量的非keep-alive的HTTP1.0请求 )，server端就会有海量TIME\_WAIT的fd的积压，有可能导致连接数耗尽而没有办法接受新的连接。

好吧，看上去它确实对我们带来了麻烦，那为什么不消灭它呢？这个家伙到底是干吗第？

## 为什么要TIME\_WAIT?

仅仅RFC里干梆梆的一句话好像不怎么make sense。来张图？那我们来看看RCF793里的Figure 13

\[code\]  
TCP A TCP B

1\. ESTABLISHED ESTABLISHED

2\. (Close)  
FIN-WAIT-1 --\> <SEQ=100\><ACK=300\><CTL=FIN,ACK\> --\> CLOSE-WAIT

3\. FIN-WAIT-2 <-- <SEQ=300\><ACK=101\><CTL=ACK\> <-- CLOSE-WAIT

4\. (Close)  
TIME-WAIT <-- <SEQ=300\><ACK=101\><CTL=FIN,ACK\> <-- LAST-ACK

5\. TIME-WAIT --\> <SEQ=101\><ACK=301\><CTL=ACK\> --\> CLOSED

6\. (2 MSL)  
CLOSED 

Normal Close Sequence

Figure 13\.  
\[/code\]  
这是一张描述连接关闭的序列图，传说中的四次握手。TCP A就是我们刚才场景里面提到的关闭请求的发起方。我们不出所料地看到了TIME\_WAIT，现在考虑如果没有这个家伙会怎样。为了描述方便，我们暂定主动关闭连接的是client端，被动关闭的是server端。当然，反过来也是一样的。

如果木有TIME\_WAIT，也就是说client在接收到server的FIN包，发出对应的ACK(对应图中的Line 4 & 5)后，就直接转入CLOSED。这时考虑如果这个ACK（Line 5）丢了的case，我们认为A是client，B是server。

a) 这时的server迟迟收不到自己FIN包的ACK，作为需要保证可靠性的TCP服务器，它会认为有可能是自己的FIN包没有到达client，而FIN包里有时是可以携带数据的。所以，server将会重发FIN包。

b) 但是别忘了，此时的client已经是CLOSED了，server会收到RST的响应，这时server断定client没有收到自己的FIN包（并且再也收不到了），但事实却并非如此------这在有时候会带来麻烦，就好比银行以为没有给你钱，但实际上你却拿走了钱。

看上去我们用反证法证明了TIME\_WAIT存在的意义。

从正面讲：如果client端发完FIN的ACK后依旧是TIME\_WAIT的话。这时如果server端重发FIN包后，client端也会相应地重新发送ACK。而client端则至多等待2\*MSL秒。

所以，TIME\_WAIT存在的理由1是：**等待足够长的时间以确保被动关闭方正常地发送出FIN包和收到ACK.**

《Unix Network Programming》的作者Richard Stevens还提到了TIME\_WAIT存在的另一个现实意义：

如果目前连接的通信双方都已经调用了close()，假定双方都到达CLOSED状态，而没有TIME\_WAIT状态时，就会出现如下的情况。现在有一个新的连接被建立起来，使用的IP地址与端口与先前的完全相同，后建立的连接又称作是原先连接的一个化身。还假定原先的连接中有数据报残存于网络之中，这样新的连接收到的数据报中有可能是先前连接的数据报。为了防止这一点，TCP不允许从处于TIME\_WAIT状态的socket建立一个连接。处于TIME\_WAIT状态的socket在等待两倍的MSL时间以后，将会转变为CLOSED状态。这就意味着，一个成功建立的连接，必然使得先前网络中残余的数据报都丢失了。

所以，TIME\_WAIT存在等理由2是：**防止上一次连接中的包，迷路后重新出现，影响新连接.**

但是在学习的过程中看到这样一个case:

> 对于大型的服务，一台server搞不定，需要一个LB(Load Balancer)把流量分配到若干后端服务器上，如果这个LB是以NAT方式工作的话，可能会带来问题。假如所有从LB到后端Server的IP包的source address都是一样的(LB的对内地址），那么LB到后端Server的TCP连接会受限制，因为频繁的TCP连接建立和关闭，会在server上留下TIME\_WAIT状态，而且这些状态对应的remote address都是LB的，LB的source port撑死也就60000多个(2^16=65536,1~1023是保留端口，还有一些其他端口缺省也不会用），每个LB上的端口一旦进入Server的TIME\_WAIT黑名单，就有240秒不能再用来建立和Server的连接，这样LB和Server最多也就能支持300个左右的连接。如果没有LB，不会有这个问题，因为这样server看到的remote address是internet上广阔无垠的集合，对每个address，60000多个port实在是够用了。
> 
> 一开始我觉得用上LB会很大程度上限制TCP的连接数，但是实验表明没这回事，LB后面的一台Windows Server 2003每秒处理请求数照样达到了600个，难道TIME\_WAIT状态没起作用？用Net Monitor和netstat观察后发现，Server和LB的XXXX端口之间的连接进入TIME\_WAIT状态后，再来一个LB的XXXX端口的SYN包，Server照样接收处理了，而不是想像的那样被drop掉了。

这个case的作者也给出了答案------《UNIX Network Programming, Volume 1, Second Edition: Networking APIs: Sockets and XTI》中提到：对于BSD-derived实现，只要SYN的sequence number比上一次关闭时的最大sequence number还要大，那么TIME\_WAIT状态一样接受这个SYN。这一点我空了在自己机器上试试看。

## TIME\_WAIT多久？

TIME\_WAIT为什么wait 2MSL呢？我们知道一个包在网络中的最长寿命是1MSL秒，乘以2是因为client要等自己的ACK去到server端，同时也要等server端可能存在的重发FIN过来自己这边。如果2MSL都没有动静，那client端就认定server端收到了自己的ACK，于是就可以安心关闭了。最坏的情况是：ACK没有发到server，并且server重试的FIN也没有发过来，这在网络正常的情况下可以视为是小概率事件，而如果它确实发生了，那么这个连接也没有存在等必要了。

不同的TCP实现有着不同的MSL定义，推荐值是1MSL=120s,但Berkeley-derived的实现却是30s，Solaris 2.x 则使用了推荐的120s，有的系统上是1min。

## 搞定TIME\_WAIT？？？？

之所以加这么多问号是因为TIME\_WAIT并不是问题，我们并不是打算完全搞定它。就象Richard Stevens说的，it's your friend and it's there to help you :-)

不同场景可以采用不同的方式，总有一款适合你。

* 减少wait的时间来减轻它的负面影响，内核也确实提供了参数来修改这个值：

\[bash\]  
net.ipv4.netfilter.ip\_conntrack\_tcp\_timeout\_time\_wait\[/bash\]

* 直接让内核快速地回收TIME\_WAIT连接：

\[bash\]\>\#让TIME\_WAIT尽快回收，我也不知是多久，观察大概是一秒钟  
echo "1"\>/proc/sys/net/ipv4/tcp\_tw\_recycle\[/bash\]

* 复用被TIME\_WAIT占用的端口：

\[bash\]\#让TIME\_WAIT状态可以重用，这样即使TIME\_WAIT占满了所有端口，也不会拒绝新的请求造成障碍  
echo "1"\> /proc/sys/net/ipv4/tcp\_tw\_reuse\[/bash\]

* 简单粗暴的方法：通过设置SO\_LINGER标志来避免socket进入TIME\_WAIT状态，这可以通过发送RST而取代正常的TCP四次握手的终止方式。够简单粗暴吧？就跟gfw对付我们一样。

但这几点其实已经违背了TCP协议设计者的初衷了，后果需要自负。但是据说windows默认就是重用TIME\_WAIT...-\_-!!!

不能重用端口可能会造成系统的某些服务无法启动，比如要重启一个系统监控的软件，它用了40000端口，而这个端口在软件重启过程中刚好被使用了，就可能会重启失败的。linux默认考虑到了这个问题，有这么个设定：

\[bash\]\#查看系统本地可用端口极限值  
cat /proc/sys/net/ipv4/ip\_local\_port\_range\[/bash\]

用这条命令会返回两个数字，默认是：32768 61000，说明这台机器本地能向外连接61000-32768=28232个连接，注意是本地向外连接，不是这台机器的所有连接，不会影响这台机器的80端口的对外连接数。如果有软件使用了40000端口监听，常常出错的话，可以通过设定ip\_local\_port\_range的最小值来解决：

\[bash\]echo "40001 61000"\> /proc/sys/net/ipv4/ip\_local\_port\_range\[/bash\]

但是这么做很显然把系统可用端口数减少了，这时可以把ip\_local\_port\_range的最大值往上调，但是好习惯是使用不超过32768的端口来侦听服务，另外也不必要去修改ip\_local\_port\_range数值成1024 65535之类的，意义不大。  
还有个有意思的参数是

\[bash\]net.ipv4.tcp\_max\_tw\_buckets\[/bash\]

表示系统同时保持TIME\_WAIT套接字的最大数量，如果超过这个数字，TIME\_WAIT套接字将立刻被清除并打印警告信息。默认为180000。

[0]: http://tools.ietf.org/html/rfc793