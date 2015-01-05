---
layout: post
title: 借助keepalived实现双master MySQL
date: 2011-11-18 15:26:23.000000000 +08:00
categories:
- 非技术
tags: []
status: draft
type: post
published: false
meta:
  _edit_last: '1'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
原则上是基本参考http://bbs.chinaunix.net/thread-1824528-1-1.html 搭的，这里主要说说搭建过程中遇到的问题，以及与之的不同之处。

## 背景

MySQL用来持久化"非永久性"数据，而不是一些形如用户信息、发表的文章之类的数据。因此不需要有专门的slave来负责被select  
采用InnoDb，因为用到很多很多借助于select for update的row lock.