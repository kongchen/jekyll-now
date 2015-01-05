---
layout: post
title: sudo on CentOS
date: 2010-06-12 10:36:10.000000000 +08:00
categories:
- 技术
tags:
- Linux
- sudo
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  dsq_thread_id: '1783232396'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
* Switch to root and install sudo first. 

\[code\]yum install sudo\[/code\]

* Edit /etc/sudoers, find below text and add your username: 

\[bash\]  
\#\# Next comes the main part: which users can run what software on  
\#\# which machines (the sudoers file can be shared between multiple  
\#\# systems).  
\#\# Syntax:  
\#\#  
\#\# user MACHINE=COMMANDS  
\#\#  
\#\# The COMMANDS section may have other options added to it.  
\#\#  
\#\# Allow root to run any commands anywhere  
root ALL=(ALL) ALL  
cartman ALL=(ALL) ALL\[/bash\]

* Now, enjoy your sudo journey!