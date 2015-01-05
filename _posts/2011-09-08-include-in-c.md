---
layout: post
title: 有趣的#include
date: 2011-09-08 16:57:02.000000000 +08:00
categories:
- 技术
tags:
- C
status: publish
type: post
published: true
meta:
  _syntaxhighlighter_encoded: '1'
  _edit_last: '1'
  _wp_old_slug: stdin
  dsq_thread_id: '1541062561'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
如果让你写一个c程序A，要求在编译时从标准输入接收另一个程序B，并且执行A的结果跟执行B一样。你一定觉得这个人是个神经病。  
好吧，就算是吧。不过难道你不觉得这挺有意思吗？  
答案是这样的------下面是A程序A.c：

\[c\]\#include "/dev/stdin"\[/c\]

编译A：

\[bash gutter="false"\]\[kongch@localhost workspace\]$ gcc A.c  
//this is program B  
\#include  
void main(){  
printf("hello\\n");  
}  
\[/bash\]

执行A:

\[bash gutter="false"\]\[kongch@localhost workspace\]$./a.out  
hello  
\[/bash\]

这个搞笑的功能要完全归功于\#include, 预编译的时候\#include 会把后面的文件完全扩展。当然，标准输入对于UNIX来说也是文件，自然也能被\#include。  
当然，你可以写一个A.c内容为\#include "B.c"，效果是一样的。

不过，这个东西有什么用呢？一个用法是在软件打build信息时，根据实际情况讲时间写入binary,例如有这样的程序when\_i\_was\_born.c：

\[c\]\#include

int main(){  
printf("%s\\n",  
\#include "/dev/stdin"  
);  
return 0;  
}\[/c\]

编译命令如下：

\[bash gutter="false"\]\[kongch@localhost workspace\]$echo "\\"\`date\`\\"" | gcc when\_i\_was\_born.c\[/bash\]

这个命令的意思是把命令date的输出作为输入交给gcc when\_i\_was\_born.c。运行下看看结果

\[bash gutter="false"\]  
\[kongch@localhost workspace\]$ ./a.out  
2011年 09月 08日 星期四 16:45:23 CST  
\[/bash\]