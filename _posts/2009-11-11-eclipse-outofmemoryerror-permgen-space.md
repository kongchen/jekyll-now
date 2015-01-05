---
layout: post
title: 'Eclipse OutOfMemoryError: PermGen space'
date: 2009-11-11 17:31:26.000000000 +08:00
categories:
- 技术
tags:
- Eclipse
- IDE
- Intellij
- Java
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  dsq_thread_id: '1546260023'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
一直都比较偏爱Intellij作为开发用的IDE。以前用破解版用惯了，它贴心的设计和强悍的代码重构功能让我爱不释手。但现在到了大公司，不得不忍痛割爱，换回抛弃了很久的Eclipse。

之所以很久没用，完全是被它的不稳定搞怕了。莫名其妙的错误窗口，变魔术一样地消失等症状很是让人抓狂。

到eclipse主页，发现eclipse已经由我记忆中的callisto变成了伽利略,飞快地down下了一个j2ee版，用了几天，感觉还很不错的说。

但好感觉仅持续了一周。该死的弹出窗口在一次正常的双击eclipse快捷方式后，再次出现了：  

[![danm eclipse error](assets/eclipse-error-300x139.png)][0]  
在cmd下执行和eclipse.exe同一文件夹的eclipsec.exe, 得到：

> Error occurred during initialization of VM  
> Could not reserve enough space for object heap

Google了一大圈，说是要把eclipse.ini中的

\[code\]--launcher.XXMaxPermSize  
256M  
\[/code\]

改成

\[code\]-XX:MaxPermSize=256m  
\[/code\]

依照着做了，也确实可以启动起来了，不过只用了一小会儿eclipse就打了一坨的error log，随后弹出个OOM的对话框，确定以后eclipse进程就消失了。  
......

又是一大圈的google，终于找到个算是解决办法的办法，试了一下确实可行，用了几天也没再crash，跟遇到类似问题的朋友分享一下，具体如下：

1. **删除eclipse.ini，或者改个其他名字**
2. **右击eclips的快捷方式，将属性中的目标改为**

\[code\]"E:\\eclipse\\eclipse.exe" -vmargs -Xms256m -Xmx1024m -XX:MaxPermSize=256m  
\[/code\]

注:引号里是eclipse.exe的位置，因人而异。

说白了，就是手动指定eclipse运行的vm参数，而不是让eclipse.exe自己去读eclipse.ini再确定。不过这样的方法不适用于在eclipse.ini中指定第三方虚拟机的方式。

[0]: http://www.kongch.com/wp-content/uploads/2009/11/eclipse-error.png