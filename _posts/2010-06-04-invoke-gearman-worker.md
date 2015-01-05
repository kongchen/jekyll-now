---
layout: post
title: 执行任意语言编写的Gearman worker/client
date: 2010-06-04 16:49:50.000000000 +08:00
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
  _syntaxhighlighter_encoded: '1'
  dsq_thread_id: '1543420469'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
![](assets/gearman64.gif)了解Gearman的同学都知道（不了解的可以点[这里][0])，对于需要开发者自行定义的Gearman client和worker来说，语言是不受限制的------只需遵循[Gearman协议][1]。并且在Gearman的官方主页上，也罗列了[各种语言实现的client和worker的API][2]。其中C的lib以及shell命令行方式的gearman是跟Gearman server绑定的，perl版本则是Gearman的前身，剩下的则绝大部分是热心网友贡献的。

你当然可以根据你自身应用来选择合适的client和worker api，它们基本都实现了Gearman定义的协议，包括job执行时状态查询(GET\_STATUS)等等。但是使用由第三方（相对于gearmand来说）开发api带来的问题也很明显：

* 它跟gearmand不是同步的；
* 你使用的api也许是一个没有被很好测试的产品
* 如果你使用的api不幸地被它的作者抛弃了（比如gearman-java），你得自己承担后续的bug fixing.

如果**你只关心结果job执行结果而不关心执行进度，同时又不愿使用一个第三方的api**，那么本文或许能对你有所帮助。

"任意语言编写的"worker、client并不是什么都是"任意"的，它还必须满足一个条件：**得是一个可执行程序**。它可以是c/cpp编出的可执行文件，也可以是一个包含main的 java类，也可以是一个perl脚本、python脚本、lua脚本...，剩下的事情就简单了。  
没错，你猜对了，我们要用的就是跟gearmand师出同门的gearman。它跟libgearman一样，包含在gearmand的包中。编译安装了gearmand之后，gearman与gearmand就会同时存在于你的机器上。我们先来看gearman的帮助，在命令行执行gearman -H后：  
\[code\]Client mode: gearman \[options\] \[\]  
Worker mode: gearman -w \[options\] \[ \[ ...\]\]

Common options to both client and worker modes.  
-f - Function name to use for jobs (can give many)  
-h - Job server host  
-H - Print this help menu  
-p  
- Job server port  
-t - Timeout in milliseconds  
-i  
- Create a pidfile for the process

Client options:  
-b - Run jobs in the background  
-I - Run jobs as high priority  
-L - Run jobs as low priority  
-n - Run one job per line  
-N - Same as -n, but strip off the newline  
-P - Prefix all output lines with functions names  
-s - Send job without reading from standard input  
-u - Unique key to use for job

Worker options:  
-c - Number of jobs for worker to run before exiting  
-n - Send data packet for each line  
-N - Same as -n, but strip off the newline  
-w - Run in worker mode\[/code\]  
嗯，说的很详细。gearman可以通过参数-w 和 -c 来指定自己是扮演worker还是client。我们要做的仅仅是利用shell命令[xargs][3]。不废话了，直接上命令：  
\[code\](gearman -h $jobservers -w -n -f $function -- xargs $bin $options) \>\> $logfile 2\>&1 &  
\[/code\]  
$jobserver是Gearman Server的地址, $function就是要注册的function名，$bin就是可执行文件的路径，$options则是你应用可能需要的参数等。, 我们还一把gearman执行时的标准输出和输入都定向到$logfile中，以便观察。

Easy enough? 但是通过这种方式启动的worker还有几点要注意：

1. worker的标准输出就是worker的返回。所以在worker的实现里需要注意像printf,System.out.print这类往标准输出里扔数据的行为，只有在worker工作完成后需要返回时，才可以调用它们。
2. 每次job被worker grab后，其实都相当于在本地新fork一个process去执行你指定的可执行程序，执行完毕后process结束。等价于直接在shell命令行敲命令，如果你的worker是对性能要求比较高的应用，shell的此番折腾也需要考虑进去。

client和worker类似，只是输入时标准输入罢了，有兴趣的同学可以自己试试，欢迎feedback。

最后，附送一个完整的用gearman启动worker的脚本，里面用到了获取cpu核数的脚本。它会启动你指定个数的worker，如果不指定，则启动与机器总core数相同的worker。  
\[bash\]\#!/bin/bash

\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#  
\# Following parameters should be configured \*  
\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#  
logfile=/tmp/gworker.log  
jobservers="127.0.0.1:4730"  
bin="/usr/local/gworker"  
options="-c conf.ini -d -l $logfile"  
function="hahaha"  
\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#  
currdir=$(dirname $0)

if \[ ! -x $bin\];then  
echo "error executable file:$bin"  
exit 1  
fi

if \[ ! -f $logfile \];then  
echo "error log file:$logfile"  
exit 1  
fi

if \[ $\# == 0 \]; then  
workers=$($currdir/cpus.sh)  
echo "using cpus cores\[$workers\] workers..."  
else  
workers=$1  
fi

i=0

while \[ $i -lt $workers \]; do  
(gearman -h $jobservers -w -n -f $function -- xargs $bin $options) \>\> $logfile 2\>&1 &  
echo "worker\[$i\] for server\[$jobservers\] started!"  
i=$(expr $i + 1)  
done

current=$(ps -ef | grep $bin | grep -v grep | wc -l)  
echo There are $current workers running now...  
\[/bash\]

[0]: http://www.kongch.com/2010/04/gearman/
[1]: http://gearman.org/index.php?id=protocol
[2]: http://gearman.org/index.php?id=download#client_worker_apis
[3]: http://en.wikipedia.org/wiki/Xargs