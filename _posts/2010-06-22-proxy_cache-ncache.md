---
layout: post
title: proxy_cache跟NCache一毛钱关系都没有？
date: 2010-06-22 10:18:28.000000000 +08:00
categories:
- 技术
tags:
- cache
- Nginx
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  dsq_thread_id: '1532653072'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
有点迷惑了，[ncache的主页][0]上头版头条写着

> ## NCache is now in nginx core , you can use it as nginx proxy cache. see [here][1][¶][2]
> 

然而我却在Nginx的邮件列表里看到[这么一条thread][3].

> \> "NCache is now in nginx core , you can use it as nginx proxy cache."
> 
> Just for the record: this statement isn't true as no parts of  
> ncache is in nginx core AFAIK; proxy\_cache is completely different  
> thing.
> 
> The one from the same page which is probably true is that "NCache is  
> out of maintaince".

说这个话的人叫Maxim Dounin，在 [http://nginx.org/en/CHANGES][4] 里看一下这个名字出现的频率就知道这个人说的话有多么靠谱了。

原本还挺为ncache感到自豪的，毕竟中国人自己的开源项目能被收录到Nginx core里实在是一件很不容易的事情。但事实看起来好像不是的。现在疑惑的是ncache怎么突然就不维护了？并且即便不维护，为何又说ncache is in nginx core now? 这话太具误导性了。

昨天大致看了下ncache的文档，好像是基于[varnish][5]设计的。不过单看介绍已经不能让我放心了，还是打算好好看看proxy\_cache和ncache的代码，再下结论。

[0]: http://code.google.com/p/ncache/
[1]: http://wiki.nginx.org/NginxHttpProxyModule#proxy_cache
[2]: http://code.google.com/p/ncache/#NCache_is_now_in_nginx_core_,__you_can_use_it_as_nginx_proxy_cac
[3]: http://forum.nginx.org/read.php?2,4979,46951#msg-46951
[4]: http://nginx.org/en/CHANGES
[5]: http://www.varnish-cache.org/