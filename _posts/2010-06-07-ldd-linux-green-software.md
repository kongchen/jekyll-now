---
layout: post
title: 利用ldd打造Linux下的绿色软件包
date: 2010-06-07 14:09:36.000000000 +08:00
categories:
- 技术
tags:
- C
- ldd
- Linux
- 绿色软件
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  _wp_old_slug: '%e5%88%a9%e7%94%a8ldd%e6%89%93%e9%80%a0linux%e4%b8%8b%e7%9a%84%e7%bb%bf%e8%89%b2%e8%bd%af%e4%bb%b6%e5%8c%85'
  _syntaxhighlighter_encoded: '1'
  dsq_thread_id: '1543315542'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
> 题记：其实这个想法来源于一个同事在之前公司的经验，感觉不错，记下来以免忘记。在云泛滥的今天，如何在云上快速部署应用其实是一个很有意思的话题。这种方法在我看来除了安装包比较大坨这一缺点之外，非常适合用来部署云应用。
> 
> 

众所周知绿色软件是windows下的一个概念，指的是仅仅一个文件夹硬copy过来就可以运行的软件，卸载时也同理：删除这个文件夹即可。说白了就是不会向系统文件夹、注册表这些地方乱丢东西的软件。

Linux下安装软件通常是编译安装，不同发行版本也有自己的二进制安装包，例如RedHat的rpm、Ubuntu的deb等。安装的最终结果往往也是向/usr/bin, /usr/lib, /usr/sbin...等诸如此类的文件夹下放上自己需要的东东。二进制包安装的话，卸载起来倒是不麻烦，但如果你是编译安装，而想卸载时又搞丢了当时的Makefile文件，那就比较讨厌了。有方法可以制作类似于Winodws下的绿色软件包吗？  

答案是肯定的。我们可以利用[ldd][0]来实现这样的功能。ldd会告诉我们对应二进制可执行程序所用到的共享库文件，我们要做的第一步是在已安装好完整应用的机器上把这些共享库文件拿到。第二步便可以在其他干净的机器上利用LD\_LIBRARY\_PATH指定运行软件时的动态库文件地址，运行对应的二进制程序。

这里我写了一个实现第一步的脚本：  
\[bash\]\#!/bin/sh

if \[ $\# -ne 1 \]; then  
echo "You should specify a dynamically executable file"  
exit 1  
fi

if \[ ! -x $1 \]; then  
echo "You should specify a executable file"  
exit 1;  
fi

path=$1

\#\#\#\#\#\#\#\#\#\#\#\#\# check file type \#\#\#\#\#\#\#\#\#\#\#\#\#  
result=$(ldd $path)

filter=$(file $path|grep dynamically|grep executable)

if \[ -z "$filter" \]; then  
echo "$path is not a dynamically executable file!"  
file $path  
exit 1  
fi

\#\#\#\#\#\#\#\#\#\#\# mkdir for .so files \#\#\#\#\#\#\#\#\#\#\#\#  
packagename=$(basename $path)-pack  
mkdir -p $packagename

IFS="  
"

\#\#\#\#\#\#\#\#\# copy .so files \#\#\#\#\#\#\#\#\#\#\#\#  
for line in $result; do  
sofile=notexitsfile  
if \[ -z "$(echo $line | grep =)" \]; then  
sofile=$(echo $line | awk '{print $1}')  
else  
\# has soft link  
sofile=$(echo $line | awk '{print $3}')  
fi

if \[ ! -f $sofile \]; then  
echo "ERROR FILE: $sofile"  
else  
echo "copying $sofile now..."  
cp $sofile $packagename  
fi  
done

cp $path $packagename

echo DONE\[/bash\]  
这个脚本的接收二进制可执行程序的路径作为参数，利用ldd找到对应的so位置，并将它们连同二进制可执行程序本身拷贝到一个文件夹中。脚本正确执行完毕后，这个文件夹就可以拿到其他的干净的机器上运行了。

举个在CentOS5.3上的例子吧，例如我在A机器上已经编译安装（或者rpm 安装）好了ffmpeg。于是乎我就可以执行我的脚本fetchsofiles.sh，以/usr/bin/ffmpeg作为参数：  
\[code\]\[root@CT53-64-BASE ~\]\# ./fetchsofiles.sh /usr/bin/ffmpeg  
copying /usr/lib64/libpostproc.so.51 now...  
copying /usr/lib64/libswscale.so.0 now...  
copying /usr/lib/libavdevice.so.52 now...  
copying /usr/lib/libavformat.so.52 now...  
copying /usr/lib/libavcodec.so.52 now...  
copying /usr/lib/libavutil.so.49 now...  
copying /lib64/libm.so.6 now...  
copying /lib64/libpthread.so.0 now...  
copying /lib64/libc.so.6 now...  
copying /usr/lib64/libz.so.1 now...  
copying /lib64/libdl.so.2 now...  
copying /lib64/ld-linux-x86-64.so.2 now...  
DONE\[/code\]  
看来是正确执行了，现在当前目录的ffmpeg-pack下就有了：  
\[code\]  
\[root@CT53-64-BASE ~\]\# ls -lh ffmpeg-pack/  
total 24M  
-rwx------ 1 root root 85K Jun 7 05:52 ffmpeg  
-rwx------ 1 root root 137K Jun 7 05:52 ld-linux-x86-64.so.2  
-rwx------ 1 root root 506K May 19 09:17 libamrnb.so.3  
-rwx------ 1 root root 392K May 19 09:17 libamrwb.so.3  
-rwx------ 1 root root 887K May 19 09:17 libasound.so.2  
-rwx------ 1 root root 4.5M Jun 7 05:52 libavcodec.so.52  
-rwx------ 1 root root 25K Jun 7 05:52 libavdevice.so.52  
-rwx------ 1 root root 647K Jun 7 05:52 libavformat.so.52  
-rwx------ 1 root root 45K Jun 7 05:52 libavutil.so.49  
-rwx------ 1 root root 67K May 19 09:17 libbz2.so.1  
-rwx------ 1 root root 1.7M Jun 7 05:52 libc.so.6<strong\></strong\>  
-rwx------ 1 root root 3.2M May 19 09:17 libdirac\_decoder.so.0  
-rwx------ 1 root root 4.5M May 19 09:17 libdirac\_encoder.so.0  
-rwx------ 1 root root 23K Jun 7 05:52 libdl.so.2  
-rwx------ 1 root root 232K May 19 09:17 libfaac.so.0  
-rwx------ 1 root root 729K May 19 09:17 libfaad.so.2  
-rwx------ 1 root root 58K May 19 09:17 libgcc\_s.so.1  
-rwx------ 1 root root 955K May 19 09:17 libmp3lame.so.0  
-rwx------ 1 root root 601K Jun 7 05:52 libm.so.6  
-rwx------ 1 root root 22K May 19 09:17 libogg.so.0  
-rwx------ 1 root root 51K Jun 7 05:52 libpostproc.so.51  
-rwx------ 1 root root 143K Jun 7 05:52 libpthread.so.0  
-rwx------ 1 root root 53K May 19 09:17 librt.so.1  
-rwx------ 1 root root 954K May 19 09:17 libstdc++.so.6  
-rwx------ 1 root root 161K Jun 7 05:52 libswscale.so.0  
-rwx------ 1 root root 208K May 19 09:17 libtheora.so.0  
-rwx------ 1 root root 1.1M May 19 09:17 libX11.so.6  
-rwx------ 1 root root 1.8M May 19 09:17 libx264.so.68  
-rwx------ 1 root root 12K May 19 09:17 libXau.so.6  
-rwx------ 1 root root 22K May 19 09:17 libXdmcp.so.6  
-rwx------ 1 root root 71K May 19 09:17 libXext.so.6  
-rwx------ 1 root root 84K Jun 7 05:52 libz.so.1\[/code\]  
24M，确实不小。我们现在就可以把这个24M的文件夹随处(\*注1)拷贝了，例如我们现在将其拷贝到从来没有安装过ffmpeg及其相关依赖的机器B上，运行LD\_LIBRARY\_PATH=/your\_pack\_path /your\_pack\_path/ffmpeg：  
\[code\]\[root@CT53-64-BASE ~\]\# cd ffmpeg-pack/  
\[root@CT53-64-BASE ffmpeg-pack\]\# LD\_LIBRARY\_PATH=. ./ffmpeg  
FFmpeg version 0.5, Copyright (c) 2000-2009 Fabrice Bellard, et al.  
configuration: --prefix=/usr --libdir=/usr/lib64 --shlibdir=/usr/lib64 --mandir=/usr/share/man --incdir=/usr/include --extra-cflags=-fPIC --enable-libamr-nb --enable-libamr-wb --enable-libdirac --enable-libfaac --enable-libfaad --enable-libmp3lame --enable-libtheora --enable-libx264 --enable-gpl --enable-nonfree --enable-postproc --enable-pthreads --enable-shared --enable-swscale --enable-x11grab  
libavutil 49.15\. 0 / 49.15\. 0  
libavcodec 52.20\. 0 / 52.20\. 1  
libavformat 52.31\. 0 / 52.31\. 0  
libavdevice 52\. 1\. 0 / 52\. 1\. 0  
libswscale 0\. 7\. 1 / 0\. 7\. 1  
libpostproc 51\. 2\. 0 / 51\. 2\. 0  
built on Nov 6 2009 19:11:04, gcc: 4.1.2 20080704 (Red Hat 4.1.2-46)  
At least one output file must be specified\[/code\]  
很幸运，运行成功了！

* 注1：所谓随处，并非任意的随处。必须是相同架构(arch)，相同内核版本的机器之间才行。因为注意到这些被拷贝的so有的是架构相关，有的直接就是内核自己的so。

[0]: http://linux.about.com/library/cmd/blcmdl1_ldd.htm