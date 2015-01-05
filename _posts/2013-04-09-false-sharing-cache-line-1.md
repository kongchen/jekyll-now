---
layout: post
title: 伪共享False Sharing（1）
date: 2013-04-09 00:28:09.000000000 +08:00
categories:
- 技术
tags:
- cache line
- 并发
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  dsq_thread_id: '1558144627'
  _wpcom_is_markdown: '1'
  _wp_old_slug: false-sharing-cache-line
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
计算机的心脏是CPU，所有的运算都由它来完成，而运算的对象------数据，则需要外部交给CPU，或者说要由CPU自己去读取。所以说运算的速度的快慢，不仅仅取决于CPU主频，也跟CPU读取数据的速度成正比。

让数据尽可能地靠近CPU------这便是所谓的缓存黄金法则。说白了就是让数据呆在离CPU较近的缓存里，因为离CPU越近的缓存，CPU访问其速度也越快。

![](assets/CPUCache.png)

上图是主存和CPU核心之间的缓存示意图（3级缓存）。越靠近CPU的缓存越快也越小。所以L1缓存很小但很快，并且紧靠着在使用它的CPU核。L2大一些，也慢一些，并且仍然只能被一个单独的 CPU 核使用。L3在现代多核机器中更普遍，比L2更大更慢，但不同的是它被单个插槽上的所有 CPU 核共享。最上方是主存，由全部插槽上的所有 CPU 核共享。  

当CPU执行运算的时候，它先去L1查找所需的数据，再去L2，然后是L3，最后如果这些缓存中都没有，所需的数据就要去主内存拿。走得越远，运算耗费的时间就越长。

下面这张表可能会让你对"访问速度"有一个更直观的印象：
**从CPU到**
**大约需要的 CPU 周期**
**大约需要的时间**

Memory
~60-80ns

QPI 总线传输  
(between sockets, not drawn)
~20ns

L3 cache
~40-45 cycles,
~15ns

L2 cache
~10 cycles
~3ns

L1 cache
~3-4 cycles
~1ns

寄存器
1 cycle

所以，尽可能地让缓存命中而不是跨越地去取数据。

## Cache Line

数据在内存中以page为单位存在，在cache里则是以cache line为单位。一个cache line是2的整数幂个连续字节，一般32-256个字节。最常见的是64字节。

java中一个long是8字节，因此一个cache line里可以安放8个long变量。所以说当你要访问一个long\[\]数组的第0个元素时，这个数组的0-7元素都被会加载到cache里，即便你此后并不需要访问1-7的元素。

而当你恰好访问完0元素就像访问1的话（比如遍历数组），很显然，你赚到了。但相反，如果你要访问的数据并不在连续的内存中时，你将无法从cache line这种整体加载中得到好处。

事实上这种整体加载非但不能让你回回都赚到，相反，在某些情况下它反倒会起到副作用。因为同一cache line上的数据是绑在一根绳上的蚂蚱。

## False Sharing

设想有这么一段代码

\[java\]  
volatile long head;  
volatile long tail;  
\[/java\]

很显然，head和tail会被加载到一个cache line中，看上去非但没什么问题，反倒很和谐。

没考虑下多线程的情况？  
![](assets/FalseSharing.png)

如果此时变量head正被Thread A写入，tail被Thread B读取。而A和B分别运行于不同的CPU核心上。

假设你的head被A更新了，那么这时cache和主存中head的值都会被更改。这会导致所有head所在的cache line都会失效，其中当然包括B使用的cache line------即便它其实并不关心head的值。  
而B此时如果需要读取tail，那么它就必须重新从主存中加载head和tail到一个cache line，即便tail没有被更改过。  
所以你应该看到了B有多么无辜，它需要为它人的所作所为承担后果。  
这还不是最糟的，更甚的情况是如果B也是写入操作，那么A和B任意一方的写操作都会让对方的cache line整体失效从而导致重新从主存加载。  
这就是传说中的[False Sharing][0].

问题在于这一切都在幕后悄然发生，没有任何编译告警提示你说你这段代码并发性能会非常低下。

如何解决这个问题呢？未完待续......

[0]: http://en.wikipedia.org/wiki/False_sharing