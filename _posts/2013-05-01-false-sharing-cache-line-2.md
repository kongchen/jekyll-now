---
layout: post
title: 伪共享False Sharing（2）
date: 2013-05-01 16:06:31.000000000 +08:00
categories:
- 技术
tags:
- cache line
- false sharing
- 并发
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  _wpcom_is_markdown: '1'
  _wpas_done_all: '1'
  dsq_thread_id: '3007821057'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
[接上一篇][0]，如何消除伪共享？

我们可以根据伪共享的成因构造一段代码，来看看伪共享对性能的影响。  
_注：代码是从 参考文章 [1][1]里得到的_

\[code lang=java\]  
public final class FalseSharing  
implements Runnable  
{  
public final static int NUM\_THREADS = 4; // change  
public final static long ITERATIONS = 500L \* 1000L \* 1000L;  
private final int arrayIndex;

private static VolatileLong\[\] longs = new VolatileLong\[NUM\_THREADS\];  
static  
{  
for (int i = 0; i < longs.length; i++)  
{  
longs\[i\] = new VolatileLong();  
}  
}

public FalseSharing(final int arrayIndex)  
{  
this.arrayIndex = arrayIndex;  
}

public static void main(final String\[\] args) throws Exception  
{  
final long start = System.nanoTime();  
runTest();  
System.out.println("duration = " + (System.nanoTime() - start));  
}

private static void runTest() throws InterruptedException  
{  
Thread\[\] threads = new Thread\[NUM\_THREADS\];

for (int i = 0; i < threads.length; i++)  
{  
threads\[i\] = new Thread(new FalseSharing(i));  
}

for (Thread t : threads)  
{  
t.start();  
}

for (Thread t : threads)  
{  
t.join();  
}  
}

public void run()  
{  
long i = ITERATIONS + 1;  
while (0 != --i)  
{  
longs\[arrayIndex\].value = i;  
}  
}

public final static class VolatileLong  
{  
public volatile long value = 0L;  
//public long p1, p2, p3, p4, p5, p6;  
}  
}

\[/code\]

上面这段代码做了如下的事情：

* 4个线程共享一个长度为4的数组，并分别负责写数组的1个元素。即0号线程写数组的\[0\]号元素，1号线程写\[1\]号...
* 4个线程是互相不干扰的，不需要任何锁来同步操作
* 每个线程都会写5亿次

## JDK6下

结果跟参考文章 [1][1] 里的一样，有没有第61行对结果影响巨大：  
![](assets/duration-300x180.png)

那么61行做了什么呢？

**它保证了每单个线程所负责的VolatileLong实例不在同一个缓存行上。** 虽然我们不能确定这些VolatileLong会布局在jvm堆中的什么位置（是否真的紧挨着）。它们是独立的对象。但是经验告诉我们同一时间分配的对象趋向集中于一块。 所以p1-p6这6\*8=48个字节再加上VolatileLong对象本身16个字节的对象头（48+16=64），我们可以让`VolatileLong`里的`value`与下一个`VolatileLong`里的`value`不在一个缓存行上。

这样4个线程各自的5亿次写操作完全是在cpu各个核的缓存中进行，期间没有缓存与内存的交换，所以性能就很高了。

## JDK7下

但是上面美妙的对比结果在JDK7下就不那么美妙了。如果你不巧手头只有JDK7，运行之后你会发现无论第61行存在与否，程序性能都非常的差。

原因是JDK7在编译时聪明地优化掉了p1-p6这些完全不会被使用的变量，那本来想被我们用作填充物的东西没了，自然数组里各个`VolatileLong`里的`value`又跑到一个缓存行里了。

只需要再添加一个方法，让JDK7认为p1-p6是有用而不做任何优化即可。  
补丁代码如下：

\[code lang=java\]  
public static long hotfix(final int index) {  
VolatileLong l = longs\[index\];  
return l.p1 + l.p2 + l.p3 + l.p4 + l.p5 + l.p6 + l.value;  
}  
\[/code\]

本人在JDK1.7.0下验证通过。

这里发现个有趣的现象：本来我在解决这个问题时加的是`return  l.p1 + l.p2 + l.p3 + l.p4 + l.p5 + l.p6`。但是发现没有任何效果，性能依旧很差。说明p1-p6虽然保留了下来，但是并没起到填充的作用。这也从另一方面证明了JVM并不是按照定义顺序在内存中放置对象。而是**定义和引用都在一起的话，在内存中被分配在一块的可能性更大些。**

当然，加上了补丁的程序在JDK6下依旧有效。

---

1. http://ifeve.com/falsesharing/ [↩][2] [↩][3]

[0]: http://www.kongch.com/2014/09/false-sharing-cache-line-1/
[1]: #fn-11370-1
[2]: #fnref-11370-1
[3]: #fnref2:11370-1