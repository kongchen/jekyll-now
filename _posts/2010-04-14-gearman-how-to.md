---
layout: post
title: 用Gearman进行分布式任务处理（一）
date: 2010-04-14 17:38:57.000000000 +08:00
categories:
- 技术
tags:
- Gearman
- Linux
- OpenSource
- 分布式
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  dsq_thread_id: '1532653865'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
____

# ![Gearman](assets/gearman64.gif)

Gearman是一款开源的通用的分布式任务分发框架，自己本身不做任何实际的工作。它可以将一个个的任务分发给其他的物理机器或者进程，以达到工作的并行运行和LB。 [有人说][0]Gearman是分布式 计算框架其实是不太准确的，因为相较于Hadoop而言，Gearman更偏重于任务的分发而不是执行。Gearman扮演的角色更像是一系列分布式进程 的神经系统。

### Gearman概述

看到Gearman的名字，最容易想到的是gear-man（齿轮男），但实际上它是Manager这个单词的一个anagram。官方 的解释是它就像manger一样光派发任务自己却不干活:)这也说明了它是一个任务派发框架，而不是一个计算框架。

> It is called Gearman because it is an anagram for Manager. Gearman, like a manager, distributes the worker to be done but does not actually do any itself.

Gearman出生于2005年，当时被LiveJournal.com用来做图片resize，纯perl实现。后来Brain Aker在2008年用C重写,接下来同时各种各样语言版本的client和worker也都逐一被贡献。目前的使用也很广泛：

* Digg: 45+servers, 400k jobs/day
* Yahoo: 60+ servers, 6M jobs/day
* Xing.com
* 金山逍遥网：http://blog.s135.com/dips/
* 我们VideoSharing的WorkerPool :)
* 同时它也是MogileFS的核心组件。

详细的项目信息可参见其主页：[http://launchpad.net/gearman][1]

### Gearman中的角色

Gearman框架中一共有三个角色：

1. Client： 提交任务的人。创建需要被执行的job然后发送给Job Server。
2. Worker： 真正干活的人。向Job Server注册然后从Job Server处拿活干。
3. Job Server：传说中的manager。接收client提交的Job，分发给相应的worker。并能在worker出现异常时重新派发job。

Gearman的工作原理如下图所示（来自Gearman主页），看上去是不是很简单？

![](assets/gearman_stack.png)

作为Job Server的gearmand是用C实现的，其最新版本可在[https://launchpad.net/gearmand][2] 找到。该项目比较健康，从2009-01-08的0.1版本发展到了目前(2010-04-19)最新的2010-04-05的0.13版本，邮件列表也保持着平均每天3-4个post的规模。伴随着gearmand一起的是相应的client和worker的lib库，同样也是C版本的。

worker和client有着各种各样的版本实现，不同的项目可以根据自身应用程序选择对应的API。这给项目的扩展带来了非常大的灵活性。

### Gearman的消息

Gearman中的消息是基于TCP的变长二进制消息：

* 请求和响应分别由Message Flag区分。这是一个4字节的结构。
* Message Type目前共有36个，详细的定义可见[http://gearman.org/index.php?id=protocol][3]。它是一个4字节的big-endian的整形。
* Data length定义了消息体的长度。它也是一个4字节的big-endian的整形。
* 消息体可由0-N个argument构成，argument间以'\\0'分隔。长度是Data Length。

请求的消息结构：

[![](assets/request1.jpg)][4]

响应的消息结构：

[![](assets/response.jpg)][5]

Message Type中定义的消息类型，在接下来Gearman Job的介绍中，大家会有具体的认识。

### Gearman中的Job

在了解Gearman的特性之前，我们有必要先搞清楚Gearman框架中的两种job。

#### Background job

顾名思义，后台执行的Job。具体的细节我们通过一张时序图可以有清晰的认识：

[![](assets/background-flow.png)][6]

由图可知，client提交完job，job server成功接收后返回JOB\_CREATED响应之后，client就断开与job server之间的链接了。后续无论发生什么事情，client都是不关心的。同样，job的执行结果client端也没办法通过Gearman消息框架获得。

如果我们想不通过额外的方法，仅使用Gearman就得到job的执行结果，就要考虑non-background job了。

#### Non-background job

这是一种相对于background job的job，它的时序图如下所示：

[![](assets/non-background-flow.png)][7]

由图可知，client端在job执行的整个过程中，与job server端的链接都是保持着的，这也给job完成后job server返回执行结果给client提供了通路。同时，在job执行过程当中，client端还可以发起job status的查询。当然，这需要worker端的支持的。

另外，从上面的时序图中，也能够清晰的看到，job 的assign都是在worker的GRAB\_JOB之后才发生的，这也印证了我们之前所说的：Job server本身并不主动下发job，job是由worker来"领取"的。

同时，我们也注意到，无论是否是哪种类型的 job，worker的工作流程都是一样的：

1. Worker通过CAN\_DO消息，注册到Job server上。
2. 随后发起GRAB\_JOB，主动要求分派任务。
3. Job server如果没有job可分配，就返回NO\_JOB。
4. Worker收到NO\_JOB后，进入空闲状态，并给Job server返回PRE\_SLEEP消息，告诉Job server:"如果有工作来的话，用NOOP请求我先。"
5. Job server收到worker的PRE\_SLEEP消息后，明白了发送这条消息的worker已经进入了空闲态。
6. 这时如果有job提交上来，Job server会给worker先发一个NOOP消息。
7. Worker收到NOOP消息后，发送GRAB\_JOB向Job server请求任务。
8. Job server把工作派发给worker。
9. Worker干活，完事后返回WORK\_COMPLETE给Job server。

值得注意的是，第6步中，Job server会给每个发送过PRE\_SLEEP消息的worker都发送NOOP 消息，哪个worker先进入到第7步，即哪个worker发送的GRAB\_JOB最先被Job server收到，那么这个job就被派发到哪个worker。这一点可以在worker端实现时利用起来，以控制任务的派发策略。也就是说，我们可以通过自定义worker端的请求策略的方式来达到自定义job分派策略的目的。

了解了消息流程，我们可以详细地探讨Gearman的特性了。

### Gearman的特性

通过Gearman分布式任务分发框架自然而然地便可以搭建一个分布式计算集群，从这点来说把Gearman称作是分布式计算框架也未尝不可，whatever，只要我们理解了原理，叫什么都无所谓啦。

Geraman宣称的[几大卖点][8]有：开源、多语言支持、富有弹性、高效、易于嵌入和无单点失败。开源和多语言支持毋须多言，其他几个特性需要分别说明一下。

#### 弹性

* 松耦合的接口和无状态的job让Gearman的扩展非常容易，只需要启动一个worker，注册到Job server集群即可。新加入的worker不会对现有系统有任何的影响。
* 同样，你可以在任何时候解雇任何数量的worker，即使那个worker正在干活。虽然这样有点不人道。Gearman不会让正在被执行的job丢失的，这一点我们前面已经做过了说明。

#### 高效

作为Gearman的核心，Job server的是用C实现的，由于只是做简单的任务派发，因此系统的瓶颈不会出在Job server上。Gearman的手册上[也有提到][9]

> a 16 core Intel machine is able to process upwards of 50k jobs per second.

#### 无单点失败

下图展示了Gearman cluster的组网方式：

![](assets/gearman_cluster.png)

* 作为使用Gearman的用户来说，Gearman的可用性就是Job Server和worker的可用性。由于worker在工作时与Job server是长连接，所以一旦worker发生异常，Job server能够迅速感知并重新派发这个异常worker刚才正在执行的工作。这保证了**正在执行的job不会由于worker的失败而丢失**。
* 对 于background job来说，Gearman支持持久化队列技术。即对于client提交的background job，Job server除了将其放在内存队列中进行派发之外，还会将其持久化到外部的持久化队列中。一旦Job server发生问题重启，外部持久化队列中的background job将会被恢复到内存中，参与Job server新的派发当中。这保证了**已提交未执行的background job不会由于Job server发生异常而丢失**。
* 而non- background job的实时状态client都是知道的，所以**对于non-background job来说，提交之后的异常都是可知的**，client端可以灵活地对job的异常进行处理。
* 我们从上图可以看到，两个Job server之间是没有连接的。也就是Job server间是不共享background job的。敏锐的你可能会提出说，通过让两个Job server指向同一个持久化队列，可以让两个Job serer互相备份。但实际上，这样是行不通的。因为Job server只有在启动时才会play back持久化队列中的background job。也就是说，Job server1如果宕机且永远不启动，Job server2一直正常运行，那么Job server1宕机前被提交到Job server1的未被执行的background job将永远都呆在持久化队列中，得不到执行。这可能是目前Gearman框架在可用性方面的一个缺陷。

### 小结

这次只是简单介绍了Gearman的基本原理，以后有机会会介绍一下Gearman的具体应用以及在应用时需要注意到的问题。可能的话我还想深入研究一下Gearman的高级应用，比如如何通过Gearman实现map-reduce；如何监控Gearman；如何让Gearman根据机器负载选择合适的worker；如何让background job能够返回结果等等。另外，扩展Gearman以增强其可用性也是比较大的一个话题，感兴趣的同学可以一起加入:)

[0]: http://blog.s135.com/dips/
[1]: http://launchpad.net/gearman "http://launchpad.net/gearman"
[2]: https://launchpad.net/gearmand
[3]: http://gearman.org/index.php?id=protocol
[4]: http://www.kongch.com/wp-content/uploads/2010/04/request1.jpg
[5]: http://www.kongch.com/wp-content/uploads/2010/04/response.jpg
[6]: http://www.kongch.com/wp-content/uploads/2010/04/background-flow.png
[7]: http://www.kongch.com/wp-content/uploads/2010/04/non-background-flow.png
[8]: http://gearman.org/index.php#introduction
[9]: http://gearman.org/index.php?id=manual:job_server