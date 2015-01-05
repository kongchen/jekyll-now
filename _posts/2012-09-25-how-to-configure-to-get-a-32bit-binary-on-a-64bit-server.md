---
layout: post
title: How to configure to get a 32bit binary on a 64bit server?
date: 2012-09-25 10:42:56.000000000 +08:00
categories:
- 技术
tags:
- configure
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  dsq_thread_id: '1532653400'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
`./configure --build=i686-pc-linux-gnu "CFLAGS=-m32" "CXXFLAGS=-m32" "LDFLAGS=-m32"```