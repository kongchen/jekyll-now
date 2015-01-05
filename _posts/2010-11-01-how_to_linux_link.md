---
layout: post
title: Linux下创建和使用动态库(.so)和静态库(.a)
date: 2010-11-01 16:18:24.000000000 +08:00
categories:
- 技术
tags:
- C
- dynamic link
- gcc
- Linux
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  dsq_thread_id: '1536880412'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
编译-链接-生成是众所周知的三部曲，所谓动态、静态都是链接这一步上的花头，举个我觉得还算形象的比喻就是：

> 动态库和静态库之于应用程序的区别就好比银行卡和现金之于消费者一样。消费者（app）需要用钱（invoke）时，如果有现金（.a），就可以直接用；否则，就得用银行卡去银行（.so）刷卡取钱。带着现金会让钱包很鼓，但消费者（app）可以很方便地使用。而银行卡(.so)只是薄薄一张卡片，加载很容易，但使用时要多绕一步，并存在刷不上钱的杯具(cannot open shared object file)。

我们试着把这个例子变成代码，首先定义

* money.h:

\[c\]\#ifndef MONEY\_H  
\#define MONEY  
void showmethemoney(int money);  
\#endif  
\[/c\]

* money.c:

\[c\]  
\#include <stdio.h\>  
void showmethemoney(int money){  
printf("$%d\\n",money);  
}  
\[/c\]  
接下来，分别把money搞成现金和银行卡。但无论怎样，link之前的compile，即编译成.o文件是必需的。

## 编译.c 为 .o

\[bash\]\[root@backup kongch\]\# ls  
money.c money.h  
\[root@backup kongch\]\# gcc -c money.c  
\[root@backup kongch\]\# ls  
money.c money.h money.o  
\[/bash\]

## 编译.o 为 .a

静态库文件名的命名规范是以lib为前缀，紧接着跟静态库名，扩展名为.a。  
我们将创建的静态库名为cash，则静态库文件名就是libcash.a。在创建和使用静态库时，需要注意这点。创建静态库用[ar][0]命令:  
\[bash\]\[root@backup kongch\]\# ls  
money.c money.h money.o  
\[root@backup kongch\]\# ar crv libcash.a money.o  
a - money.o  
\[root@backup kongch\]\# ls  
libcash.a money.c money.h money.o  
\[/bash\]

## 编译.o 为 .so

动态库文件名命名规范和静态库文件名命名规范类似，也是在动态库名增加前缀lib，但其文件扩展名为.so。  
我们将创建的动态库名为bank，则动态库文件就是libbank.so。用gcc来生成动态库：  
\[bash\]\[root@backup kongch\]\# gcc -shared -fPIC -o libbank.so money.o  
/usr/bin/ld: money.o: relocation R\_X86\_64\_32 against \`a local symbol' can not be used when making a shared object; recompile with -fPIC  
money.o: could not read symbols: Bad value  
collect2: ld returned 1 exit status  
\[/bash\]  
我在CentOS 5.3 x85\_64上得到了这样的错误，至于错误原因及什么是fPIC，参见[这里][1]。我们先fix这个问题再说------在编译.o的过程中使用-fPIC参数：  
\[bash\]\[root@backup kongch\]\# gcc -c -fPIC money.c  
\[root@backup kongch\]\# gcc -shared -fPIC -o libbank.so money.o  
\[root@backup kongch\]\# ls -la  
total 32  
drwx------ 2 root root 4096 Nov 1 15:04 .  
drwx------ 3 root root 4096 Nov 1 14:53 ..  
-rwx------ 1 root root 5848 Nov 1 15:04 libbank.so  
-rw------- 1 root root 1656 Nov 1 14:56 libcash.a  
-rw------- 1 root root 79 Nov 1 14:41 money.c  
-rw------- 1 root root 69 Nov 1 14:38 money.h  
-rw------- 1 root root 1552 Nov 1 15:04 money.o  
\[/bash\]  
可以看到我们有了libcash.a和libbank.so，万事俱备，只欠东风。东风就是搞段代码来调用showmethemoney.

* main.c

\[c\]\#include "money.h"

int main(){  
showmethemoney(100);  
return 0;  
}  
\[/c\]

## 静态链接

告诉gcc静态链接库在哪里就可以了：  
\[bash\]\[root@backup kongch\]\# gcc -o cash main.c -L. -lcash  
$100  
\[root@backup kongch\]\# ls -la  
total 44  
drwx------ 2 root root 4096 Nov 1 15:11 .  
drwx------ 3 root root 4096 Nov 1 14:53 ..  
-rwx------ 1 root root 6879 Nov 1 15:11 cash  
-rwx------ 1 root root 5848 Nov 1 15:04 libbank.so  
-rw------- 1 root root 1656 Nov 1 14:56 libcash.a  
-rw------- 1 root root 73 Nov 1 15:10 main.c  
-rw------- 1 root root 79 Nov 1 14:41 money.c  
-rw------- 1 root root 69 Nov 1 14:38 money.h  
-rw------- 1 root root 1552 Nov 1 15:04 money.o  
\[/bash\]  
执行cash：  
\[bash\]\[root@backup kongch\]\# ./cash  
$100  
\[/bash\]  
我们注意到cash的文件大小是6879字节。

## 动态链接

方式几乎与静态链接一样，为了区别，我们把动态链接的输出叫做bank。  
\[bash\]  
\[root@backup kongch\]\# gcc -o bank main.c -L. -lbank  
\[root@backup kongch\]\# ls -la  
total 52  
drwx------ 2 root root 4096 Nov 1 15:15 .  
drwx------ 3 root root 4096 Nov 1 14:53 ..  
-rwx------ 1 root root 6955 Nov 1 15:15 bank  
-rwx------ 1 root root 6879 Nov 1 15:11 cash  
-rwx------ 1 root root 5848 Nov 1 15:04 libbank.so  
-rw------- 1 root root 1656 Nov 1 14:56 libcash.a  
-rw------- 1 root root 73 Nov 1 15:10 main.c  
-rw------- 1 root root 79 Nov 1 14:41 money.c  
-rw------- 1 root root 69 Nov 1 14:38 money.h  
-rw------- 1 root root 1552 Nov 1 15:04 money.o  
\[/bash\]  
我们注意到动态链接的bank并不比静态链接的cash小，这好像跟前面提到的鼓钱包论矛盾了。个人猜测是我们的money.o太过简单，只有一个showmethemoney函数，体现不出.so的优势来。

试着执行一下bank：  
\[bash\]  
\[root@backup kongch\]\# ./bank  
./bank: error while loading shared libraries: libbank.so: cannot open shared object file: No such file or directory  
\[/bash\]  
失败了。这就需要我们搞清楚动态链接、执行时搜索路径顺序:

1. 编译目标代码时指定的动态库搜索路径；
2. 环境变量LD\_LIBRARY\_PATH指定的动态库搜索路径；
3. 配置文件/etc/ld.so.conf中指定的动态库搜索路径；
4. 默认的动态库搜索路径/lib；
5. 默认的动态库搜索路径/usr/lib。

我们通过2、3两种方式来让我们的bank正常：

* 方式2

在执行前手动指定so所在路径，但就像你知道的，这样的指定是seesion级别的，每次执行都要这么来：  
\[bash\]  
\[root@backup kongch\]\# LD\_LIBRARY\_PATH=. ./bank  
$100  
\[/bash\]

* 方式3

手动指定/etc/ld.so.conf里包含的路径，比如我的测试server上  
\[bash\]  
\[root@backup kongch\]\# cat /etc/ld.so.conf  
include ld.so.conf.d/\*.conf  
\[/bash\]  
那我就可以创建一个自己的conf：/etc/ld.so.conf.d/kong.conf，文件内容就是当前libbank.so所在目录：/home/kongch/  
下来要记得ldconfig，这样bank就可以工作了：  
\[bash\]  
\[root@backup kongch\]\# ldconfig  
\[root@backup kongch\]\# ./bank  
$100  
\[/bash\]  
这种方式一劳永逸，不过需要有root权限。

废话这么多，只是因为好记性不如烂笔头。更详细，更准确的使用和描述，还是参见[这里][2]吧。

[0]: http://sourceware.org/binutils/docs/binutils/ar.html#ar
[1]: http://www.gentoo.org/proj/en/base/amd64/howtos/index.xml?part=1&chap=3
[2]: http://www.yolinux.com/TUTORIALS/LibraryArchives-StaticAndDynamic.html