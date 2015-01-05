---
layout: post
title: 'AC_CHECK_LIB的工作原理以及usr/bin/ld: cannot find -lxxx的处理'
date: 2012-09-26 11:36:00.000000000 +08:00
categories:
- 技术
tags:
- AC_CHECK_LIB
- ld
- link
- Linux
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  dsq_thread_id: '1533832979'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
### 背景

了解GNU Build System那套  
\[bash\]  
./configure && make && make install  
\[/bash\]把戏的同学都知道configure脚本是用来判断当前系统相关依赖并生成Makefile的，但这篇文章并不介绍这套build system如何把玩，如果对这个感兴趣，可以从[http://www.ibm.com/developerworks/cn/linux/l-makefile/][0]看起。下面这个图就是借于其处：

![图 2生成Makefile流程图](assets/image002.gif)

由上图可知，configure文件是autoconf根据configure.in生成的。而AC\_CHECK\_LIB就是告诉configure我们需要检查哪些lib，并根据检查结果做什么处理。正是在这里遇到了问题并有所收获，才促使我写这篇blog记录一下。

问题是这样，我想检查一下libXv.so这个库是否存在，不存在的话就让configure直接退出。你可能要问人家是so你为啥要在编译阶段检查一个动态库是否存在，答案是因为代码里**静态链接**了libXv.so里的某些function.

于是乎，我在configure.in里写了如下代码：

\[code\]  
....  
AC\_CHECK\_LIB(Xv, XvGetVideo , \[\], \[  
echo "Error! You need to have libXv installed!"  
exit -1  
\])  
....  
\[/code\]

* AC\_CHECK\_LIB的第一个参数是库名，我们这里依赖libXv，自然是填Xv。
* 第二个参数是函数名，填的要么是你代码里真正reference到的function；如果你不知道或者懒得去查代码的话，用nm -D查看这个so，随便挑一个出来填上即可。
* 第三个参数是如果check结果为yes，做什么操作。我想继续，所以啥也不填。
* 第四个参数代表如果check结果为no，做什么处理。你也看到了------我打印一条报错信息，然后exit。

看上去没什么问题，我也由此生成了configure，但是在执行configure时却有了如下的报错：

\[code\]  
checking for XvGetVideo in -lXv... no  
\[/code\]

没道理啊！我是用_yum install libXv_安装的Xv库呀，而且XvGetVideo 这个名字都是通过

\[bash\]  
nm -D /usr/lib64/libXv.so.1  
\[/bash\]

得到的啊。

路径问题？不能啊，我/usr/lib和/usr/lib64下面都有libXv.so.1啊。虽然我确信我这个64bit的机器用的只会是/usr/lib64\.

好在configure执行期间会输出config.log，这也顺便让我知道了AC\_CHECK\_LIB的工作原理。

打开config.log，找到下面出错的片段：

\[code\]  
configure:4415: checking for XvGetVideo in -lXv  
configure:4440: gcc -o conftest -g -O2 conftest.c -lXv \>&5  
/tmp/ccFbn6ak.o: In function \`main':  
/home/kongc/workspace/haohao/conftest.c:23: undefined reference to \`XvGetVideo'  
collect2: ld returned 1 exit status  
configure:4440: $? = 1  
configure: failed program was:  
|  
| /\* Override any GCC internal prototype to avoid an error.  
| Use char because int might match the return type of a GCC  
| builtin and then its argument prototype would still apply. \*/  
| \#ifdef \_\_cplusplus  
| extern "C"  
| \#endif  
| char XvGetVideo ();  
| int  
| main ()  
| {  
| return XvGetVideo ();  
| ;  
| return 0;  
| }  
configure:4449: result: no

\[/code\]

原来AC\_CHECK\_LIB这个宏是生成了一个.c程序，在里面调用我们指定的函数，再让gcc编译这个.c程序通过-l去link我们指定的库，如果link成功，则代表check通过。

而我们之所以没有通过check，log里也给出了原因：_ld returned 1 exit status_

ld哪里出错了呢？为了找到原因，我手动创建了这个test.c:

\[c\]  
\#ifdef \_\_cplusplus  
extern "C"  
\#endif  
char XvGetVideo ();  
int  
main ()  
{  
return XvGetVideo ();  
;  
return 0;  
}  
\[/c\]

然后执行

\[bash\]  
gcc -g -O2 test.c -lXv  
\[/bash\]

这下得到了ld失败的真正原因：

\[code\]/usr/bin/ld: cannot find -lXv\[/code\]

路径问题？？No，刚才已经否定了这个因素了。

软连接问题？？No. 我ls看了，_/usr/lib64/libXv.so.1 -\> libXv.so.1.0.0_ 一点问题都没有，两个文件都好好的。

就连_ldconfig -p_也显示libXv被成功load了啊啊！

上下求索了好久，最终在[stackoverflow][1]找到了答案：  
对于一个so来说，在程序执行时被**动态link**，这时link对象的名字叫做SONAME，对于一个so来说它的SONAME可以通过objdump看到：

\[bash\]  
$ objdump -p /usr/lib64/libXv.so.1 | grep SONAME  
SONAME libXv.so.1  
\[/bash\]

这个SONAME的用途：是程序在执行时，在LD\_LIBRARY\_PATH下查找这个SONAME，找到对应的so文件，再进行**动态**绑定执行。[这篇说绿色软件的文章][2]依照的就是这个原理。  
这也是为何我用yum安装libXv，/usr/lib和/usr/lib64下面各被安置了一个libXv.so.1的原因。

所以你也看到了，我一直在说执行和动态链接。显然调用gcc时并不是在执行，也不是在动态链接。对于ld来说，-l{libname} 这个参数只会让ld去系统库目录，也就是/usr/lib或/usr/lib64下以及通过-L指定的目录下去找lib{libname}.so这个文件，然后静态链接。如果找不到就会报usr/bin/ld: cannot find -lxxxxxd 的错误！而我的/usr/lib64/和/usr/lib下面都是没有libXv.so这个文件的。

所以解决办法很简单，创建一个/usr/lib64/libXv.so 的软连接指向/usr/lib64/libXv.so.1即可。

上面那个stackoverflow的帖子里同时也提供了另一个解决方案，那就是安装对应库的devel包。对于libXv来说，就是安装libXv-devel。装好后的效果其实是一样：

\[bash\]  
$ yum install libXv-devel  
$ ls -la /usr/lib64/libXv.so  
lrwxrwxrwx 1 root root 14 09-25 13:45 /usr/lib64/libXv.so -\> libXv.so.1.0.0  
\[/bash\]

虽然有点杀鸡用牛刀的感觉，但也很好解释了devel阶段的link跟使用时的link的区别------说白了就是对于so来说，静态链接和动态链接时使用的名字不一样。静态链接使用libxxxx.so，动态连接使用SONAME。

### 总结

说的东一句西一句貌似没有什么主题，但这次折腾让我明白了：

1. AC\_CHECK\_LIB的工作原理
2. 对于一个so来说ld时使用的名字和执行时被动态绑定时，使用的名字是不一样的
3. 对于一个技术问题，80%中文搜索结果都是重复且无用的（况且我用的还是google）
4. 对于技术问题，更快得到更好答案的方式可能是search in stackoverflow rather than in google. 当然，case by case:)

[0]: http://www.ibm.com/developerworks/cn/linux/l-makefile/
[1]: http://stackoverflow.com/questions/335928/ld-cannot-find-an-existing-library
[2]: /2010/06/ldd-linux-green-software/