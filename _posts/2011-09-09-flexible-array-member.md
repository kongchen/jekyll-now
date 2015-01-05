---
layout: post
title: flexible array member
date: 2011-09-09 14:51:19.000000000 +08:00
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
  dsq_thread_id: '1569298415'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
学无止境，这话一点儿也不假。

被问了个小问题，涉及到flexible array member。发现这玩意儿以前还没真用心留意过：

\[c\]  
typedef struct{  
int i;  
int buf\[\];  
}s\_t;  
\[/c\]

sizeof(s\_t)等于几？答案是4.结构体最末端的char buf\[\]就是所谓的flexible array member.  
这玩意儿是C99引入的新feature,所以从可移植性上来讲并不推荐。C99对其的定义如下：

> 作为特例，一个含有多个命名成员的结构体的最后一个成员可以是不完整类型的数组，这就叫flexible array member（弹性数组成员）。这种成员有三个特点：
> 
> 1. 含有弹性数组成员的结构体，其大小等于其弹性数组成员的偏移量。
> 2. 当使用`.`或者`->`运算符引用最后的弹性数组成员时，弹性数组的大小就是这个结构体被分配内存时所指定的尺寸所能容纳的最大数组（如果不是正好容纳，那么向下取整/截断）
> 3. 如果在分配内存时为弹性数组预留的空间连区区一个数组元素也装不下，那么这时数组大小（等价）等于1，**但是！**你不能对这第一个元素进行访问（行为是未定义 的），但取其指针是合法的。
> 

听着挺绕，看例子可能就比较清晰了。对应于第一条的例子：

\[c\]  
assert(sizeof(s\_t) == offsetof(s\_t,buf));

\[/c\]

第二条的例子：

\[c\]  
s\_t \* s1 = malloc(sizeof(s\_t)+20)); //等价于struct{ int i; int buf\[5\];}s1;  
s\_t \* s2 = malloc(sizeof(s\_t)+6));//等价于struct{ int i; int buf\[1\];}s1;  
\[/c\]

第三条的例子：  
\[c\]  
s\_t \* s1 = malloc(sizeof(s\_t)+2));  
/\*等价于struct{ int i; int buf\[1\];}s1; 但是s1-\>buf\[0\]不可被access\*/  
int\* p =&(s1-\>buf\[0\]); //合法  
\*p = 2;//未定义！！  
printf("%d",\*p);//未定义！！  
\[/c\]  
**但是，**我在Linux CT53-64-BASE 2.6.18-194.11.3.el5用gcc version 4.1.2 20080704 (Red Hat 4.1.2-50)测试这里却没有报错。不知道为啥，先挖个坑，以后知道原因了再填上。