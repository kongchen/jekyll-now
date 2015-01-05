---
layout: post
title: 给Nginx添加自定义监控模块
date: 2011-08-27 11:38:31.000000000 +08:00
categories:
- 技术
tags:
- Nginx
- 共享内存
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  dsq_thread_id: '1536510668'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
需求是为Nginx添加一个能够主动汇报统计信息的功能，例如请求成功数，请求失败数，总请求数，总上行字节数，下行字节数，当前连接数等等，需要nginx实时报告给监控中心，以用于查看、监控和分析。  
上述有些统计项是Nginx本身自带的status模块就有的，我们称之为原生统计项。但这个监控模块是被动的，仅在客户端HTTP GET查询时返回而并不主动地向监控server汇报。

要想做到主动汇报，我们一定不能把这个汇报过程纳入Nginx的主cycle中，否则由于汇报过程中的网络调用block了工作进程就非常糟糕了。

顺其自然的想法就是起一个新的进程作为Agent，向监控中心汇报。而Nginx只需要让Agent知道自身的status数据就可以了。当然，你可以让Agent间隔性的请求Nginx的status接口，得到结果后上报给监控中心。但这么做的劣势也是显而易见的：1）查询请求跟其他HTTP请求无异，占用了Nginx的可用连接、内存池等资源；2）只能获得原生统计项。

所以，更进一步的想法是利用共享内存作进程间的通信。在提供原生status数据的同时，支持添加自定义的统计数据。  
之前恰好实现了这样一个模块，这里记录一下，提供一种思路。  
要实现这样的想法，主要需要考虑如下几点：

* 共享内存的注册和释放
* 定义一套可扩展的结构，作为通过共享内存传递status的载体
* 原生status可采用定时汇报的方式（定时写共享内存）
* 个性化统计项通过提供配置命令（如在nginx.conf中利用if，在请求失败/成功时将某某数据项加1）在配置文件里实现灵活配置

例如，配置文件的片段可以是：

\[code\]  
location /login {  
if($arg\_agent == 'ie'){  
status\_long\_inc "AGENT\_IE" 1;  
}  
if($arg\_agent == 'firefox'){  
status\_long\_inc "AGENT\_FF" 1;  
}  
status\_string\_set "LAST\_LOGIN\_TIME" $time;  
}  
\[/code\]

意思是：在客户端如果是ie时，统计项AGENT\_IE加1,；客户端是firefox时，统计项AGENT\_FF加1；并且更新统计项LAST\_LOGIN\_TIME为当前时间。  
**具体的思路是**：

* master启动注册共享内存；
* 由某一worker负责将原生status定时写入共享内存；
* 个性化统计项的命令处理handle里对共享内存进行更新。

但同时注意到一些Nginx的特性：

* Nginx中各个worker process间是对等关系
* 每一个worker进程都可以读写原生status信息
* 每个worker都需要操作注册的共享内存，写前加锁，写后释放
* 每个worker都是被master监控的，一旦异常退出，master会随即起一个新的worker

这里想要引出的意思是：

我们只需要一个worker process来专门负责汇报原生status信息就可以了。设想每个worker都来定时地往共享内存写原生的status信息，那过多的加解锁势必会对性能造成影响。所以得有一个专门负责汇报原生status的worker，我们暂且叫它worker leader。这个worker leader在启动的时候就要确定好领导地位，同时注意到Nginx的worker是'可杀的'，因为master发现某个（些）worker被杀，会自动再spawn一个（些）。实现时要注意worker leader被杀的case。

**具体实现**：

主要是ngx\_module\_s这个结构里的几个hook要实现：

\[c\]  
struct ngx\_module\_s {  
ngx\_uint\_t ctx\_index;  
ngx\_uint\_t index;

ngx\_uint\_t spare0;  
ngx\_uint\_t spare1;  
ngx\_uint\_t spare2;  
ngx\_uint\_t spare3;

ngx\_uint\_t version;

void \*ctx;  
ngx\_command\_t \*commands;  
ngx\_uint\_t type;

ngx\_int\_t (\*init\_master)(ngx\_log\_t \*log);

ngx\_int\_t (\*init\_module)(ngx\_cycle\_t \*cycle);

ngx\_int\_t (\*init\_process)(ngx\_cycle\_t \*cycle);  
ngx\_int\_t (\*init\_thread)(ngx\_cycle\_t \*cycle);  
void (\*exit\_thread)(ngx\_cycle\_t \*cycle);  
void (\*exit\_process)(ngx\_cycle\_t \*cycle);

void (\*exit\_master)(ngx\_cycle\_t \*cycle);

uintptr\_t spare\_hook0;  
uintptr\_t spare\_hook1;  
uintptr\_t spare\_hook2;  
uintptr\_t spare\_hook3;  
uintptr\_t spare\_hook4;  
uintptr\_t spare\_hook5;  
uintptr\_t spare\_hook6;  
uintptr\_t spare\_hook7;  
};  
\[/c\]

**init\_master**  
共享内存的注册需要放在这里面，以保证只会被注册一次，在master起来的时候注册，worker在被fork的时候也拿到了读写权。但是最初我发现并不能成功。再仔细看了nginx的代码，发现master init这个hook是无效的，可能是作者考虑这个钩子函数如果开放风险太大吧，毕竟这个地方如果有什么闪失，整个Nginx就起不来了。所以可能需要给Nginx打个patch：

\[patch\]

diff -Naur src/os/unix/ngx\_process\_cycle.c src-kongch/os/unix/ngx\_process\_cycle.c  
--- src/os/unix/ngx\_process\_cycle.c 2010-09-15 23:24:21.000000000 +0800  
+++ src-kongch/os/unix/ngx\_process\_cycle.c 2011-01-24 10:11:31.000000000 +0800  
@@ -131,6 +131,15 @@  
ngx\_setproctitle(title);

+ /\*\*\*\*\*\*START of init\_master hook//by kongch\*\*\*\*\*\*/  
+ for (i = 0; ngx\_modules\[i\]; i++) {  
+ if (ngx\_modules\[i\]-\>init\_master) {  
+ if(ngx\_modules\[i\]-\>init\_master(cycle-\>log) != NGX\_OK)  
+ ngx\_master\_process\_exit(cycle);  
+ }  
+ }  
+ /\*\*\*\*\*\*END of init\_master hook//by kongch\*\*\*\*\*\*/  
+  
ccf = (ngx\_core\_conf\_t \*) ngx\_get\_conf(cycle-\>conf\_ctx, ngx\_core\_module);

ngx\_start\_worker\_processes(cycle, ccf-\>worker\_processes,

\[/patch\]

**init\_process**

在这里面就可以做刚才提到的选worker leader了。由于走到这里的时候，各个worker进程已经是相互独立的了，他们之间要选一个出来当老大肯定是需要交流的，进程间交流？还是通过共享内存吧，我的做法是通过锁，让所有worker都去试图去将自己的pid写入共享内存。当然也有判断所谓的leader是否还真活着的逻辑，具体如下图所示:  
[![](assets/worker-leader.jpg)][0]

整个模块完整的实现由于并不具有通用性（和agent绑死），所以也没必要开源了。思路在这里，有兴趣的可以将agent合入nginx并将之通用化:)

[0]: http://www.kongch.com/wp-content/uploads/2011/08/worker-leader.jpg