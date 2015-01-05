---
layout: post
title: Dynamo琐碎
date: 2011-08-31 14:23:51.000000000 +08:00
categories:
- 技术
tags:
- dynamo
- 分布式
- 架构
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  dsq_thread_id: '1532654673'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
这篇可以看做是看[http://www.slideshare.net/iso1600/key-value-store][0] 的一个笔记，也算是一个静下心来仔细学习dynamo的记录。

**[分布式Key Value Store漫谈][1]** 

View more [presentations][2] from [Tim Y][3] 

Dynamo的论文[有哥们翻成过中文][4]，可参阅。

其中心思想有三：最终以执行；始终可写；去中心化。

要理解这三点需要的预备知识有：

## CAP理论

CAP是三个词的缩写：

* Consistent 一致性------ 说白了，就是读出来的跟写进去的要一致
* Availability 可用性------ 所有操作的都能返回成功
* Partition tolerance 隔离容忍度------ 分布式系统内部两组server间一旦不能传递信息，系统是否还能正常工作？

通常对于CAP的一句话总结就是：三选二。但是也看到有不同的意见------[http://www.cloudera.com/blog/2010/04/cap-confusion-problems-with-partition-tolerance/ ][5]

Dynamo的选择是AP。牺牲的是一致性。

## 一致性哈希 consistent hashing

分布式可不仅仅是服务端的事情，客户端也需要作相应的配合。这篇讲一致性哈希的文章讲得很细也很容易理解------[http://blog.csdn.net/sparkliang/article/details/5279393][6]

## Quorom NRW

同样也有地儿可寻------[http://ultimatearchitecture.net/index.php/2010/06/22/quorum-nwr/][7]

> N：同一份数据的Replica的份数  
> W：是更新一个数据对象的时候需要确保成功更新的份数  
> R： 读取一个数据需要读取的Replica的份数

NWR值的不同组合会产生不同的一致性效果:

> <N,W,R\>=<1,1,1\>和单点运行的数据库是同一个配置。
> 
> <N,W,R\>=<2,1,1\>则相当于Slave-Master模式。由于1+1不大于2，所以这种情况是可能读到非最新数据的。也就是这种配置是不一致的。
> 
> W越大，写性能越差。R越大，读性能越差。N越大，数据可靠性就越强。为了保障一致性，平衡读写性能，通常的配置是：W=Q, R=Q ，Q=N/2+1（N=3，R=2，W=2的配置就满足这个公式）。

btw: 我看[InfoQ上有个对小米科技米聊团队同学的采访][8]，提到米聊采用的是NRW是211。个人感觉这样的可靠性很不强啊,不过还是要看业务场景。个人使用米聊的感觉是没有用户主动地更新和删除操作，这可能也是他们敢211的原因吧。

Dynamo的选择是322。

## Vector Clock

之前学习过：[http://www.kongch.com/2011/08/vector-clock-understanding/][9]

## gossip

也找到了一篇中文文章来学习[http://blog.csdn.net/chen77716/article/details/6275762][10]

Gossip算法如其名，灵感来自办公室八卦，只要一个人八卦一下，在有限的时间内所有的人都会知道该八卦的信息，这种方式也与病毒传播类似，因此Gossip有众多的别名"闲话算法"、"疫情传播算法"、"病毒感染算法"、"谣言传播算法"。

但Gossip并不是一个新东西，之前的泛洪查找、路由算法都归属于这个范畴，不同的是Gossip给这类算法提供了明确的语义、具体实施方法及收敛性证明。

Gossip是一种去中心化、容错而又最终一致性的绝妙算法，其收敛性不但得到证明还具有指数级的收敛速度。使用Gossip的系统可以很容易的把Server扩展到更多的节点，满足弹性扩展轻而易举。

## Hinted handoff

Hinted handoff是用来处理系统短暂的失效的方法，当所有主要负责的 N个节点均失效的情况下，它试图将信息存放在非主要责任节点的一个**特殊的位置**，并记下一个 hint,其包含这次写操作的真正目标节点信息。当消息服务收到一个 Gossip信号得知有新的节点从失败中恢复过来时，它查看该节点该节点有没有需要移交 (handoff)的数据。通过检查它是否是那个 hint提到的节点，如果是，包含 hint的那个节点将向它移交 (handoff) replica.

## Merkle Tree

Merkle Tree, 又被称为[Hash Tree][11]，是一种树状Hash结构，1979年由Ralph Merkle发明。

在分布式系统中，它被巧妙地用来进行节点接数据一致性的检查。 介绍可参阅：[http://ultimatearchitecture.net/index.php/2010/09/12/merkle-tree/][12]

[0]: http://www.slideshare.net/iso1600/key-value-store
[1]: http://www.slideshare.net/iso1600/key-value-store "分布式Key Value Store漫谈"
[2]: http://www.slideshare.net/
[3]: http://www.slideshare.net/iso1600
[4]: http://hjpetstore.googlecode.com/files/Amazon's%20Dynamo%20中文.pdf
[5]: http://www.cloudera.com/blog/2010/04/cap-confusion-problems-with-partition-tolerance/
[6]: http://blog.csdn.net/sparkliang/article/details/5279393
[7]: http://ultimatearchitecture.net/index.php/2010/06/22/quorum-nwr/
[8]: http://www.infoq.com/cn/interviews/cc-internet-distributed-system-architecture
[9]: http://www.kongch.com/2011/08/vector-clock-understanding/
[10]: http://blog.csdn.net/chen77716/article/details/6275762
[11]: http://en.wikipedia.org/wiki/Hash_tree "Merkle Tree"
[12]: http://ultimatearchitecture.net/index.php/2010/09/12/merkle-tree/