---
layout: post
title: 'dot: command not found'
date: 2011-08-09 11:06:47.000000000 +08:00
categories:
- 技术
tags:
- google
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  dsq_thread_id: '1546300578'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
I tried to use [google performance tool][0] to analyze my app's memory usage, but got the error: "sh: dot: command not found".  
The reason is that I did not install graphviz Graph Visualization Tools, after installed it, the error went away.  
\[shell\]  
yum install graphviz -y  
\[/shell\]

[0]: http://code.google.com/p/google-perftools/wiki/GooglePerformanceTools