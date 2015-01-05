---
layout: post
title: 同步方法（synchronized method）与同步块（synchronized block）
date: 2014-09-19 23:32:59.000000000 +08:00
categories:
- 技术
tags:
- synchronized
- 并发
status: publish
type: post
published: true
meta:
  _wpcom_is_markdown: '1'
  _edit_last: '1'
  _wpas_done_all: '1'
  dsq_thread_id: '3026901806'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
### 发现

`synchronized`用在方法签名当中和用于代码块有什么不同吗？今天在观察另一个问题时无意中发现两者的差别。看下面的代码：

\[code lang=java\]  
public class SyncSample {  
private String s;

public synchronized String get1() {  
return s;  
}

public String get2() {  
synchronized( this ) {  
return s;  
}  
}  
}  
\[/code\]

`get1()`和`get2()`在实际工作时效果是一样的，但字节码却有着很大的差别。  
`get1()`的字节码如下：

\[code lang=text\]  
public synchronized java.lang.String get1();  
descriptor: ()Ljava/lang/String;  
flags: ACC\_PUBLIC, ACC\_SYNCHRONIZED  
Code:  
stack=1, locals=1, args\_size=1  
0: aload\_0  
1: getfield \#2 // Field s:Ljava/lang/String;  
4: areturn  
\[/code\]

而`get2()`的：

\[code lang=java\]  
public java.lang.String get2();  
descriptor: ()Ljava/lang/String;  
flags: ACC\_PUBLIC  
Code:  
stack=2, locals=3, args\_size=1  
0: aload\_0  
1: dup  
2: astore\_1  
3: monitorenter  
4: aload\_0  
5: getfield \#2 // Field s:Ljava/lang/String;  
8: aload\_1  
9: monitorexit  
10: areturn  
11: astore\_2  
12: aload\_1  
13: monitorexit  
14: aload\_2  
15: athrow  
\[/code\]

### 分析

仔细对比`get1()`与`get2()`,发现前者在flags里多了一个`ACC_SYNCHRONIZED`，字节码是给JVM看的，JVM看到这个flag就知道进入这个方法是需要获取对象的锁，离开（包括遇到异常离开）后释放。

而`get2（）`则是用了很多的篇幅来告诉JVM应该怎样进入和离开同步块。

分析到这里我一度认为前者性能会好些，但转念一想，字节码只是在JVM**加载时**用的，最终执行还是要转成机器码由机器来执行。那这两种方式对最终执行的性能有何影响吗？

我做了个实验，分别运行`get1()`和`get2()` 1亿次，运行所耗时间几乎相等。

### 结论

字节码是被JVM读取的，JVM在加载类的时候通过读取类的字节码知道如何执行类中的方法。上述两种方法只是在字节码上有所差异，只是`get1()`更多是让JVM自动地去做了一些事情，最终运行是没有差异的。

想到一个个人感觉比较贴切的比喻：

> `get1()`的字节码是自动档，而`get2()`的字节码是手动档，虽然前者看着简单，最终都逃不掉齿轮扣齿轮带动轮胎转动这个步骤，速度并不会因为是自动还是手动而有很大差异。
>