---
layout: post
title: keepalived的log
date: 2011-11-22 14:20:18.000000000 +08:00
categories:
- 技术
tags:
- keepalived
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  dsq_thread_id: '1532654468'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
keepalived读配置文件遇到错误是不给任何提示的，这个往往让初用着摸不着头脑------明明配了xxx，怎么不起作用呢？------比如说关键字TCP\_CHECK后面得有个空格才能写{，否则就是不生效！这个蛋疼的设定让我抓狂了一个上午。  
有个方法能够缩短抓狂的时间，就是利用keepalived的启动参数：  
\[bash\]keepalived -D -S 0 -d"\[/bash\]

> -D是让keepalived详细记录log  
> -d是在日志里打出keepalived读到的配置信息  
> -S 0是到会将日志发送给syslog，且日志定为LOCAL0

这里需要让syslog帮忙把接收到的日志写到文件以便我们查看：加一行配置到syslog.conf，然后重启syslog服务。  
\[bash\]  
echo "local0.\* /var/log/keepalived.log"\>\> /etc/syslog.conf  
/etc/init.d/syslog restart  
\[/bash\]  
这样，你就可以通过查看/var/log/keepalived.log来检查你是否成功地使用了keepalived了。  
例如我有这样的配置  
\[text\]  
MISC\_CHECK {  
misc\_path "/root/pingM.sh"  
\# misc\_timeout 5  
}  
\[/text\]  
启动时下面这段log就反映了我的配置，而如果我配置文件中的MISC\_CHECK之后不加空格，这段信息是绝对不会出现的。  
\[text\]  
Nov 22 01:13:52 localhost Keepalived\_healthcheckers: ------< Health checkers \>------  
Nov 22 01:13:52 localhost Keepalived\_healthcheckers: 10.224.178.164:3306  
Nov 22 01:13:52 localhost Keepalived\_healthcheckers: Keepalive method = MISC\_CHECK  
Nov 22 01:13:52 localhost Keepalived\_healthcheckers: script = /root/pingM.sh  
Nov 22 01:13:52 localhost Keepalived\_healthcheckers: timeout = 0  
Nov 22 01:13:52 localhost Keepalived\_healthcheckers: dynamic = NO  
\[/text\]