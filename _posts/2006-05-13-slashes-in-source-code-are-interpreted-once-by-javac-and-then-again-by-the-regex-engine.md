---
layout: post
title: slashes in source code are interpreted once by javac and then again by the
  regex engine
date: 2006-05-13 13:14:00.000000000 +08:00
categories:
- 技术
tags:
- Java
- trick
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  dsq_thread_id: '1532652931'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
今天用string.replaceAll("\\\\n","\\n");本义是想将字符串里的字符串\\n转换成字符\\n,哪知道却没效果,心想可能是转义字符有问题,于是试着改成replaceAll("\\\\",test")想看看是啥状况,哪知却出现了Unexpected internal error near index 1的错误.google了一下,居然在sun的[_Bug Database_][0]里面找到了,不过已经被定性为 Closed, not a bug.**Evaluation** 里写道:This is not a bug. As mentioned in the spec, **slashes in source code are interpreted once by javac and then again by the regex engine**. 

也就是说,对于replaceAll("\\\\",test"),\\\\先被javac解释成\\,然后交给正则引擎,正则引擎看到\\后认为这是一个转义字符,于是去找后面的字符,却发现没有了,然后报错.string.replaceAll("\\\\n","\\n") doesn't work的原因也很好解释了,\\\\n到了正则引擎处就成了\\n(换行符)了,而不是\\n

知道原因就好办了,改成string.replaceAll("\\\\\\\\n","\\n"); //it works!

[0]: http://spaces.msn.com/bugdatabase/