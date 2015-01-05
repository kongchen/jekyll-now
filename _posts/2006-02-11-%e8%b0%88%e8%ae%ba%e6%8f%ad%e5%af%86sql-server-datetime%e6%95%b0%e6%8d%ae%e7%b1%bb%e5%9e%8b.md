---
layout: post
title: 谈论揭密SQL Server DATETIME数据类型
date: 2006-02-11 11:59:00.000000000 +08:00
categories:
- 技术
tags:
- datetime
- SQL Server
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  dsq_thread_id: '1532652212'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
揭密SQL Server DATETIME数据类型

看完这篇文章的第一感觉是，虽然对于日期类型数据使用得很算顺利，不过作者 提到的一些东西还真不知道。有时候在应用上，不觉得比老外差到那里去。但是， 老外的一个优良习惯细扣概念并进行实证检验；而我们的习惯是概念是概念，应用 是应用。到最后会发现其实有些很基础的东西，是不知其所以然的。 

---

原文：[Demystifying the SQL Server DATETIME Datatype][0]  
来源：[SQL-Server-Performance.com][1]  
作者：Frank Kalis

When you follow online communities dedicated to SQL Server with open eyes, you certainly notice...... 

---

你和发现网上很多SQL Server的问题是关于DATETIME数据类型的，这似乎说明熟练使用DATETIME并不容易。 

奇怪的是，我却一直相信使用DATETIME是不难的事。DATETIME并非复杂的数据类型，也没有深奥的日期算法。唯一需要 理解的是为了安全的处理临时数据，DATETIME数据类型的一些基本概念。本文的目的就是帮助读者理解这些SQL Server有趣 的地方，以及弄清楚DATETIME数据类型的一些真相。 

本文我都会使用ISO日期格式 yyyymmdd。这是一种安全的日期格式，即无论你的电脑如何设置，该格式都可以运行正常，而且 它也不受SET DATEFORMAT或者SET LANGUAGE设置的影响。即使你不开发国际用户的数据库应用，也最好养成使用安全的 日期格式的习惯。SQL Server只有两种日期类型格式编号是安全的，112和116。112是ISO格式，116是ISO8601格式。 在SQL Server联机帮助的CAST和CONVERT主题中可以找到关于这两种日期编号的介绍。你越早养成这些习惯，很多潜在的 问题就越少。 

你将注意到，我特意使用了隐示地将CHAR转换成DATETIME。隐示转换有时候不是一种良好的开发习惯，不过根据 SQL Server数据类型中，DATETIME转换的优先级高，我认为转换是安全的。关于这点本文就不多阐述了。 

本文先来研究一下DATETIME数据类型的内部表现形式，然后将注意力转移到DATETIME相关的查询上，最后总结一些 注意事项，小技巧和常见问题的解决方法。 

所有的代码示例都适用于SQL Server 2000，我相信对于以前的SQL Server版本也应该适用。对于2000以后的版本，我将 检查本文并对不适用的地方作相应的改动。

好了，让我们开始吧！ 

DATETIME类型的数据的可读性

SELECT CAST(GETDATE() AS BINARY(8)) AS WhatIsReallyStored

WhatIsReallyStored  
------------------  
0x00009620016CE3A8

(1 row(s) affected)

我必须承认，我对16进制数并不那么精通，所以我想我无法直接告诉你这串数字表示的是 2005-03-23 22:08:31.280\. ;-) 

SQL Server联机帮助(SQL Server Books Online :**BOL**)的解释是 "DATETIME数据类型的值储存在2个4byte长度的整数中。" 

说明就只有这些。 

**注意**，BOL并没有说DATETIEM类型的值是保存在某种特定的日期或者时间格式中，而这些格式是依赖于 语言和计算机设置的! 而仅仅是说储存在2个整型数值中。这并非100％的描述清楚了。其实是这样的， 该值的确是储存在2个4byte长度的整型中，但是被打包成了BINARY(8)。前4个字节保存和19000101这个日期的 差值，我们知道SQL Server的基准日期是19000101。 

后4个字节保存时间数据，该数据是以午夜开始的累积毫秒数。 

这些也能在BOL中找到相关的介绍。下面我们用一个简单的脚本来检验。 

DECLARE @300 BINARY(8)  
SET @300 = 0x00000000 + CAST(300 AS BINARY(4)) 

在讨论运行结果前，我们先来看一下该脚本干什么用。首先定义了一个SQL Server DATETIME内部数据类型。 然后我们将前4个byte(即日期部分)赋值为0x00000000。这意味着我们将它设置为基准日期(=19000101)。 然后在日期部分后连接一个时间部分。时间部分赋值为300，根据BOL，我们知道，这表示300毫秒，即 1/3秒。运行脚本 

SELECT  
@300  
, CAST(@300 AS DATETIME)

------------------ ------------------------------------------------------  
0x000000000000012C     1900-01-01 00:00:01.000

(1 row(s) affected)

奇怪，我们的结果显示是1秒，而不是300毫秒。反过来推理，赋值300表示1秒，也就是说它并不是 表示300毫秒，而是300/300秒，即1秒。 

实际上，SQL Server的确储存从午夜开始的时钟。每个时钟单位是3.33毫秒。这也就是为什么DATETIME 数据类型最小精确度是1/300。在BOL关于也是如此介绍的。 

DATETIME数据类型的基本查询

本节我们将利用SQL Server自带的Northwind示例数据库。 

DATETIME查询和其它数值查询方式是一样的。也就是说，你可以使用数值类型可以使用的比较操作，如 =, <\>, \>= or <=。而且也可以使用逻辑操作，如 LIKE 或 BETWEEN。下面进行示例。 

Example 1: 检索条件为特定时间： 

SELECT  
CustomerID  
, OrderDate  
FROM  
Orders  
WHERE  
OrderDate = '19960704'

CustomerID OrderDate  
---------- ------------------------------------------------------  
VINET    1996-07-04 00:00:00.000

(1 row(s) affected) 

**注意**，这里有一个使用DATETIME最重要的地方。既然DATETIME类型既包括日期部分，也包括时间部分， 查询中就要考虑这两个部分。幸运的是，Northwind Orders数据表只保存日期信息在OrderDate列中。为了 说明问题，我们修改查询如下： 

UPDATE  
Orders  
SET  
OrderDate = '19960704 12:31:00'  
WHERE  
CustomerID='VINET'  
AND  
OrderDate = '19960704'  
GO

CustomerID OrderDate  
---------- ------------------------------------------------------

(0 row(s) affected) 

避免该情况唯一可做的就是在WHERE语法后同时指定时期和时间。然而在大多数情况下，我们并 不知道确切的时间或者只是对日期感兴趣。 

Example 2: 检索条件为阶段性时间

上面的例子中我们想把所有日期为'19960704'的记录查找出来，但是没有成功。下面我们给出有效的办法。 

SELECT  
CustomerID  
, OrderDate  
FROM  
Orders  
WHERE  
OrderDate \>= '19960704'  
AND  
OrderDate < '19960705'

CustomerID OrderDate  
---------- ------------------------------------------------------  
VINET    1996-07-04 12:31:00.000

(1 row(s) affected) 

上面是满足需求的最简单办法。基本思路就是你指定大于'19960704'(从凌晨00:00:00开始)，小于 '19960705'(到午夜00:00:00结束)的时间段，搜索该时间段内的记录。这样处理，你就不必去做 分割时间格式等工作，上面的查询还可以改写为： 

SELECT  
CustomerID  
, OrderDate  
FROM  
Orders  
WHERE  
OrderDate  
BETWEEN  
'19960704'  
AND  
'19960705'

CustomerID OrderDate  
---------- ------------------------------------------------------  
VINET   1996-07-04 12:31:00.000  
TOMSP   1996-07-05 00:00:00.000

(2 row(s) affected) 

需要注意的是，上面的写法也许会造成一点麻烦，因为BETWEEN会搜索包括结束日期在内的记录。 BETWEEN的确是这样搜索的，比如我们改写为： 

SELECT  
CustomerID  
, OrderDate  
FROM  
Orders  
WHERE  
OrderDate  
BETWEEN  
'19960704'  
AND  
'19960704'

CustomerID OrderDate  
---------- ------------------------------------------------------

(0 row(s) affected) 

这样，查询就无效了。如果不显式设置，SQL Server缺省将时间设置为午夜(00:00:00)。 因此，上面的查询无法找出时间不为午夜的任何记录。前面三个例子中，第一个例子是安全的， 推荐使用。无论采取那种方式，首要的是要搞清楚我们想要的到底是什么。 

顺便提一下，上面查询在执行计划中，SQL Server会自动将BETWEEN解析为 \>= 和 <= 的方式。因此，无法查询出记录也就不奇怪了。在执行计划中，解析的信息如下： 

...SEEK:(\[Orders\].\[OrderDate\] \>= Convert(\[@1\]) AND \[Orders\].\[OrderDate\] <= Convert(\[@2\]))... 

上面例子中，我们实际用到了 \>= 和 <= ，所以也不用专门给出这两个操作的例子了。 我们来看一下LIKE操作符的用法。BOL中指出LIKE可以用于搜索日期和时间部分，基本用法如下： 

SELECT  
CustomerID  
, OrderDate  
FROM  
Orders  
WHERE  
OrderDate  
LIKE  
'%1996%'

CustomerID OrderDate  
---------- ------------------------------------------------------  
VINET   1996-07-04 12:31:00.000  
TOMSP   1996-07-05 00:00:00.000  
HANAR   1996-07-08 00:00:00.000  
...

在这个例子中，我们要查找1996年的记录。现在我们扩展一下需求，查找1996年7月份的所有记录。也许有人会用 LIKE '%1996-07%'的WHERE条件。然而运行这种查询后，检索不到任何记录。为了叙述简单，我只是建议大家不要 局限于DATETIME的LIKE操作用法。当处理日期或者时间的一部分时，SQL Server提供了一个更方便的内置函数，上面 的例子可改写为： 

SELECT  
CustomerID  
, OrderDate  
FROM  
Orders  
WHERE  
DATEPART(yyyy,OrderDate)=1996

CustomerID OrderDate  
---------- ------------------------------------------------------  
VINET   1996-07-04 12:31:00.000  
TOMSP   1996-07-05 00:00:00.000  
HANAR   1996-07-08 00:00:00.000  
...

利用DATEPART函数，可以很容易对月份进行查询。 

SELECT  
CustomerID  
, OrderDate  
FROM  
Orders  
WHERE  
DATEPART(yyyy,OrderDate)=1996  
AND  
DATEPART(mm,OrderDate)=7

CustomerID OrderDate  
---------- ------------------------------------------------------  
VINET   1996-07-04 12:31:00.000  
TOMSP   1996-07-05 00:00:00.000  
HANAR   1996-07-08 00:00:00.000  
...

我还是建议使用Example 2的方式，如： 

SELECT  
CustomerID  
, OrderDate  
FROM  
Orders  
WHERE  
OrderDate \>= '19960701'  
AND  
OrderDate < '19960801' 

DATETIME高级查询

本节中要讨论的是，当对DATETIME查询时，我们是否应该多选择DATEADD()或者DATEDIFF()函数。 实际上，DATEADD()，DATEDIFF()和\>=,<=并没有多大的区别。假设OrderDate字段有聚集索引，那么 这两种方式得到的性能结果基本是一样的。不过，我还是建议多使用DATEADD函数。原因下面阐述。 

假设你想要知道从某天开始起，经过7天后的那个日期的订单情况，查询大致如下： 

DECLARE @dt DATETIME  
SET @dt = '19960701'  
SELECT  
OrderID  
, CustomerID  
FROM  
Orders  
WHERE  
OrderDate = DATEADD(DAY,7,@dt)

OrderID CustomerID  
----------- ----------  
10250   HANAR  
10251   VICTE

(2 row(s) affected) 

既然OrderDate字段上有索引，那么执行计划显示为：

...|--Index Seek(OBJECT:(\[Northwind\].\[dbo\].\[Orders\].\[OrderDate\])...

SQL Server当然会用到该字段的索引，然后再看看I/O情况

Table 'Orders'. Scan count 1, logical reads 6, physical reads 0, read-ahead reads 0\.

然后我们看看DATEDIFF函数的效果： 

SELECT  
OrderID  
, CustomerID  
FROM  
Orders  
WHERE  
DATEDIFF(d,@dt,OrderDate)=7

OrderID CustomerID  
----------- ----------  
10250   HANAR  
10251   VICTE

(2 row(s) affected) 

当然查询结果是一样的，那么执行计划如何呢？

...|--Clustered Index Scan(OBJECT:(\[Northwind\].\[dbo\].\[Orders\].\[PK\_Orders\])...

显然，SQL Server无法使用OrderDate字段的索引，因此只有扫描整个表才能给出查询结果。从I/O性能 上我们能得出相同的推断：

Table 'Orders'. Scan count 1, logical reads 21, physical reads 0, read-ahead reads 0\.

使用DATEDIFF的逻辑读取次数是使用DATEADD的3倍多! 

当然，Orders表只有830条记录，因此无论那种方式运行都很快。但是如果记录很多时，情况就不一样了， 至少我肯定愿意使用DATEADD函数。 

如何在当前日期加上(减去)N天?

正确的回答是使用DATEADD函数，例如： 

DECLARE @dt DATETIME  
SET @dt = '20050325'  
SELECT  
DATEADD(d,1,@dt)

------------------------------------------------------  
2005-03-26 00:00:00.000

(1 row(s) affected) 

其实，既然SQL Server's的基础日期单位是天，因此也可以： 

SELECT  
@dt+1

------------------------------------------------------  
2005-03-26 00:00:00.000

(1 row(s) affected) 

两种方式是等价的，都对某日期加上N天。如果是减去N天，把N改成-N天就行了。 

这种处理同样适用于天数的分割。假设你要对上面的日期加上2个小时，可以如下使用： 

DECLARE @dt DATETIME  
SET @dt = '20050325'  
SELECT  
DATEADD(hh,2,@dt)

------------------------------------------------------  
2005-03-25 02:00:00.000

(1 row(s) affected) 

或者写为： 

SELECT  
@dt+0.08333333333333333

------------------------------------------------------  
2005-03-25 02:00:00.000

(1 row(s) affect 

从上面的分析中，我们可以知道为什么选择用DATEADD函数。因为它比较容易被理解，而 也不必将2个小时具体换算成天为单位，例如 2小时＝2/24天＝0.08333333333333333天。 

为什么可以用 DATEADD(d, DATEDIFF(d, 0...))的方法来去掉时间部分对结果的影响?

准确的说，SQL Server 2000以及以前的版本，DATETIME数据类型总是包含两部分：日期和时间。你不能 把这两部分分割开。所以用"去掉"这个词可能会造成误解。其实我们是把时间部分设置为午夜时间，从而 来避免错误的发生。下面是其中的一种用法： 

SELECT  
DATEADD(d,DATEDIFF(d,0,GETDATE()),0)

------------------------------------------------------  
2005-03-23 00:00:00.000

(1 row(s) affected) 

我们来解剖该语句，看看它为什么能去掉时间影响。首先 

SELECT  
DATEDIFF(d,0,GETDATE())

-----------  
38432

(1 row(s) affected) 

这句查询是关键! 

DATEDIFF函数返回两个日期间的天数差，如： 

SELECT  
DATEDIFF(d,'20050228 23:59:59.997', '20050301 00:00:00.000')

-----------  
1

(1 row(s) affected) 

当然没有人会认为上面那两个时间差是1天。但是，既然DATEPART参数是d，DATEDIFF函数就只会考虑 日期部分，下面的语句和上面的查询实际上效果是一样的。 

SELECT  
DATEDIFF(d,'20050228', '20050301')

-----------  
1

(1 row(s) affected) 

因此无论实际上两个时间有多接近，你最后得到的结果总是1天。回到上面的例子，DATEDIFF返回基准日期 和当前日期之间的天数。太棒了，这样我们就可以得到想要的结果，而不必去处理时间部分。我们只需要 将DATEDIFF得到的结果放入DATEADD函数中去计算即可。 

SELECT  
DATEADD(d,DATEDIFF(d,0,GETDATE()),0)  
, DATEADD(d,38432,0)

------------------------------------------------------ ------------------------------------------------------  
2005-03-23 00:00:00.000   2005-03-23 00:00:00.000

(1 row(s) affected) 

当写本文时，我觉得这真是简单。 

SELECT  
CAST(DATEDIFF(d,0,GETDATE()) AS DATETIME)

------------------------------------------------------  
2005-03-23 00:00:00.000

(1 row(s) affected) 

上面的例子也能很好的达到目的。 

正如你所看到的，这不涉及任何复杂的日期或者数值算法。坦率地说，这其实很简单。而且，对于DATEDIFF 和DATEADD函数的任何参数来说，道理是一样的。 

最后提一下，我们还可以利用DATETIME的内部存储格式来设置一个DATETIME类型的时间部分为午夜时间，如： 

SELECT  
CAST(SUBSTRING(CAST(GETDATE() AS BINARY(8)),1,4) + 0x00000000 AS DATETIME)  
, CAST(CAST(SUBSTRING(CAST(GETDATE() AS BINARY(8)),1,4) AS INT) AS DATETIME)

------------------------------------------------------ ------------------------------------------------------  
2005-03-23 00:00:00.000   2005-03-23 00:00:00.000

(1 row(s) affected) 

[0]: http://www.sql-server-performance.com/fk_datetime.asp
[1]: http://www.sql-server-performance.com/