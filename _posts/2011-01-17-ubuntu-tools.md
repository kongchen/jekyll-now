---
layout: post
title: 有了它，基本可以脱离windows了
date: 2011-01-17 23:30:16.000000000 +08:00
categories:
- 技术
tags:
- 我的电脑
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  dsq_thread_id: '1543310916'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
![](assets/images.jpg)今天较困，原因是昨晚熬到2点多看了中国队的比赛。刚刚眯着眼睛做了下面这些事情：

1，按照[这个说明][0]装上了kscope 需要说明的是原文中的脚本有个链接失效了，我用的脚本是[fix\_kscope.sh.tar][1] 事实证明Ubuntu 10.10也是可以work的。（这个与本篇主题无关，只是蓄谋已久而已------Linux下的SourceInsight）

2，按照[这个说明][2]装上了'附瑞给特'，当然借助了wine，其中装wine的要点是：

> 3 部分 DLL 设置  
> 在真实的 windows 系统中从 C:\\WINDOWS\\systenm32 里复制 mfc42.dll,msvcp60.dll, riched20.dll,riched32.dll 这几个文件到 /home/用户名/.wine/drive\_c/windows/system32 文件里，需要覆盖时确定。其他dll文件不要随便覆盖，要做备份。  
> 4 字体设置  
> 从 Windows 目录下的 Fonts 里的 simsun.ttc 复制到 /home/user/.wine/drive\_c/windows/fonts 里面。  
> 把下面的代码保存为 zh.reg ，然后终端执行 regedit zh.reg 。

这些文件打包放在...[forwine.tar][3]

3，按照[这个说明][4]，给Chromium配上了Proxy Switchy。要点是：

> 1. 打开设置，首先创建一个 Proxy Profile，命名后输入地址和端口并保存。
> 2. 进入 Switch Rules 选项卡，设置 gfwlist 的 URL 为 [http://autoproxy-gfwlist.googlecode.com/svn/trunk/gfwlist.txt][5] 。记得要勾选 "AutoProxy Compatible List"，然后代理选择刚才命名的，你知道的。
> 

4，把VirtualBox里xp的漏洞都补掉了。万一哪天要用也能用用。

就这样了，睡了。

[0]: http://blog.solrex.org/articles/install-kscope-on-ubuntu-9-04.html
[1]: http://www.kongch.com/wp-content/uploads/2011/01/fix_kscope.sh.tar.gz
[2]: http://gothefirst.appspot.com/?p=101001
[3]: http://www.kongch.com/wp-content/uploads/2011/01/forwine.tar.gz
[4]: http://blog.xiao3.info/chrome-switchy-autoproxy-gfwlist-pac.html
[5]: http://autoproxy-gfwlist.googlecode.com/svn/trunk/gfwlist.txt