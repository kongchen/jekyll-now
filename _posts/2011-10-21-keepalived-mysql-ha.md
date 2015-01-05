---
layout: post
title: 改良版本的使用keepalived构建高可用mysql-HA
date: 2011-10-21 14:11:28.000000000 +08:00
categories:
- 技术
tags:
- ha
- keepalived
- mysql
- 架构
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  dsq_thread_id: '1532655441'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
最近在折腾一个任务调度系统。作为企业级应用的一部分，HA is a must.

作为一个HA的任务调度系统，丢任务自然是不允许的。因此需要将已提交的任务持久化。MySQL是个比较容易想到的持久化容器。同时注意到HA的要求------No single point of failure。MySQL也不能例外。于是就有了今天这个笔记。

MySQL要做到HA,复制是必须的。且failover后要能继续服务，自然得考虑多master的架构。多master下，数据一致性很重要，而数据一致性的保证完全取决于复制。因此决定使用正式引入了semi-sync replication机制的MySQL GA版本：5.5.17。 最起码在复制的可靠度上能有所保障------虽然不是100%保证。(MySQL的semi-sync我也只是初接触，后面可以好好看看学习学习)

但是复制只能保证数据的高可用，服务的高可用需要采取另外的手段。所谓服务高可用说白了就是保证客户端能持续地使用MySQL。即便一台MySQL故障，那么在短暂的失败时间内，客户端也可以通过重连来切换到正常的MySQL上，继续刚才的工作。

能想到的最简单方案就是两台MySQL互为master-slave。由MySQL的semi-sync复制来保证数据的连续性。另外考虑到这个系统的特殊性：

1. 用MySQL是来持久化"非永久性"数据的(任务做完了就没必要再留着了)
2. 借助于select for update实现的lock（防止任务被重复调度、更新）

基于以上两点，这个MySQL-HA方案就不需要传统的多slave用于查询的架构了。但同时也要求所有的客户端在同一时刻只能去访问一台MySQL来操作数据。MySQL Cluster和开源的MMM应该都可以满足我上述的需求。但我大致扫了一眼，觉得有点大------比我这个任务调度系统本身还要大，有点本末倒置，而且很复杂------越复杂越容易出错，所以就没有深入看下去。碰巧找到[使用keepalived构建高可用mysql-HA][0]这篇guide，简单易操作，符合我的口味。这里主要说说搭建过程中遇到的问题，以及我加的一点优化。

根据上面的叙述，我们有了下面的拓扑。本文的叙述也都是基于这个拓扑来的：

> 访问数据库的VIP:**10.224.178.5**
> 
> MySQL-A: **10.224.178.164**
> 
> MySQL-B: **10.224.178.167**

![](assets/QQ%E6%88%AA%E5%9B%BE%E6%9C%AA%E5%91%BD%E5%90%8D1.jpg)

两台MySQL都安装keepalived.

\[text\]\# yum install kernel-devel  
\# wget http://www.keepalived.org/software/keepalived-1.2.1.tar.gz  
\# tar zxvf keepalived-1.2.1.tar.gz  
\# ./configure --with-kernel-dir=/usr/src/kernels/2.6.18-274.7.1.el5-x86\_64/\[/text\]

这里选用了keepalived-1.2.1。configure时需要指定内核src目录,这样就可获得IPVS Framework的支持。（没用当前最新的1.2.2的原因是加了--with-kernel-dir后编译不通过，而1.2.1已经可以满足需求了）

![](assets/QQ%E6%88%AA%E5%9B%BE%E6%9C%AA%E5%91%BD%E5%90%8D.jpg)

keepalived在这个架构中负责两件事：1、VIP的维护； 2、MySQL的healthcheck。

只需要简单的配置文件即可完成这两个任务的设置。

我们自己新建一个配置文件，默认情况下keepalived启动时会去/etc/keepalived目录下找配置文件.

\[text\]  
mkdir /etc/keepalived  
vi /etc/keepalived/keepalived.conf  
\[/text\]

这是10.224.178.164上的配置文件

\[text highlight=""0,7,10,12,30,31,34""\]\#Configuration File for keepalived  
global\_defs {  
router\_id MySQL-ha  
}

vrrp\_instance VI\_1 {  
state BACKUP \#两台配置此处均是BACKUP  
interface eth0  
virtual\_router\_id 51  
priority 100 \#优先级，另一台改为90  
advert\_int 1  
nopreempt \#不抢占，只在优先级高的机器上设置即可，优先级低的机器不设置  
authentication {  
auth\_type PASS  
auth\_pass 1111  
}  
virtual\_ipaddress {  
10.224.178.5  
}

}  
virtual\_server 10.224.178.5 3306 {  
delay\_loop 2 \#每个2秒检查一次real\_server状态  
lb\_algo wrr \#LVS算法  
lb\_kind DR \#LVS模式  
persistence\_timeout 60 \#会话保持时间  
protocol TCP  
real\_server 10.224.178.164 3306 {  
weight 1  
notify\_down /root/when\_db\_down.sh \#检测到服务down后执行的脚本  
notify\_up /root/when\_db\_up.sh

MISC\_CHECK {  
misc\_path "/root/pingM.sh"  
misc\_timeout 5  
}  
}  
}  
\[/text\]

另一台机器10.224.178.167上的配置也类似，这里就不贴了，按照上面的注释可以很容易的得到。

这里讲下跟引文的区别：

1. 用自己写的MISC\_CHEK脚本，而不是简单的TCP\_CHECK
2. 检测到服务down后的脚本
3. 检测到服务startup的脚本。

引文中，服务down后的脚本做的工作是直接pkill keepalived。这样直接杀死了keepalived从而使得当前机器的VIP不再对外，从而client访问VIP也就到达不了这台down了的机器上来，新的请求会被另一台VIP开启的机器接管。

这样比较麻烦的一件事情就是当down掉的MySQL重新起来后，还需要在那台机器上手动将keepalived启动，才能再次开启VIP和healthcheck。有人说本来down掉的MySQL就需要手动来启动的啊，只是多做件事情罢了。但是考虑如果MySQL由于过于繁忙，导致healthcheck脚本的连接超时而被认为down了。这种情况无需人工干预MySQL可以自行恢复，那么我们的keepalived最好也能自行恢复。

keepalived强大的启动参数可以帮助我们来实现这一点，但也需要我们写脚本来配合。我在这里做了些改动，主要有四个脚本。

### 1.startMySQL.sh 用来启动MySQL服务。

\[bash\]  
\#!/bin/sh  
/etc/init.d/mysql start  
keepalived -D -d -S 0 -f /etc/keepalived/keepalived.conf --vrrp\_pid /var/run/vrrp.pid  
\[/bash\]

启动MySQL都用这个脚本。它做了两件事情：1、启动mysqld;2、启动keepalived。并且将vrrp进程的pid写入/var/run/vrrp.pid中------这个很关键。

### 2.pingM.sh 用来监控MySQL服务

\[bash\]\#!/bin/sh  
mysql -u root -ppass -N -h 127.0.0.1 -e "select 1;" databasename\>\>/dev/null  
exit $?\[/bash\]

如果db正常，则返回0.否则返回1.keepalived的判断完全基于这个返回码。

当然，我们可以很灵活地更加个性化地写自己的MySQL监控程序来加入这个脚本，让keepalived的监控更加精确。

### 3.when\_db\_down.sh 当MySQL down时触发的脚本

\[bash\]\#!/bin/sh  
if \[ -f /var/run/vrrp.pid \]; then  
pkill keepalived  
ip -s -s a f to 10.224.178.5/32  
\#only run health check when mysql goes down  
keepalived -C -D -d -S 0 -f /etc/keepalived/keepalived.conf  
fi\[/bash\]

这里就用到了我们之前写的vrrp.pid文件。我们判断这个文件的存在性，如果存在则关闭整个keepalived服务，然后再单独开启keepalived的healthcheck进程，用来检查MySQL是否又启动了。

这个判断的作用在于，如果MySQL一直未恢复，那么healthcheck进程依旧会定时的触发when\_db\_down.sh脚本，这时由于vrrp.pid不存在，因而不会做任何事情。免掉了healthcheck进程被pkill杀死又启动这样的麻烦事。

\[text\]ip -s -s a f to 10.224.178.5/32\[/text\]

的作用在于手动删除VIP，因为我有时发现keepalived被杀死后，VIP还在ip table中，手动删除确保VIP不再对外。

### 4.when\_db\_up.sh 当MySQL恢复后触发的脚本

\[bash\]\#!/bin/sh  
pkill keepalived  
/root/startMysql.sh\[/bash\]

杀死当前的healthcheck进程，然后再用脚本1启动MySQL服务。

经过测试，客户端访问10.224.178.5:3306。首先链接机器A，当机器A的MySQL宕机后。客户端会出现connection lost的exception。但3s-5s后重试，便可成功连接到机器B。而当机器A上MySQL恢复后，keepalived自动再次启用VIP，但客户端连接依旧保持在B上------这点很重要！只有当B上的MySQL挂了，连接才会跟之前一样，切换到A上。否则B就一直作为master对外服务。

两台机器其实不分主备，因为都有可能承担全部的服务请求，所以硬件配置上没有孰优孰劣之说。

[0]: http://bbs.chinaunix.net/thread-1824528-1-1.html