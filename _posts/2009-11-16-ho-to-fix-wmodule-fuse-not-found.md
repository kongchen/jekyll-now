---
layout: post
title: 'How to fix "FATAL: Module fuse not found." on CentOS 4'
date: 2009-11-16 16:47:02.000000000 +08:00
categories:
- 技术
tags:
- CentOS
- FUSE
- Linux
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  _wp_old_slug: how-to-fix-module-fuse-not-found
  _syntaxhighlighter_encoded: '1'
  dsq_thread_id: '1532651959'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
![FUSE](assets/fuse.png)  
\[code\]\> cat /etc/\*-release  
\[/code\]  
will tell you your CentOS's version.

First, get rmpforge from [HERE][0] according to your OS version.  
then,  
\[code\]\> rpm -ivh rpmforge-release-$version.$arch.rpm\[/code\]  
CentOS 4  
\[code\]\> yum install kernel-smp-devel dkms dkms-fuse\[/code\]  
CentOS 5  
\[code\]\> yum install dkms dkms-fuse\[/code\]  
Then,  
\[code\]\> dkms remove -m fuse -v 2.7.4-1.nodist.rf --all  
\> dkms add -m fuse -v 2.7.4-1.nodist.rf  
\> dkms build -m fuse -v 2.7.4-1.nodist.rf --kernelsourcedir=/usr/src/kernel/yourkenerlsource  
\> dkms install -m fuse -v 2.7.4-1.nodist.rf  
\> modprobe fuse\[/code\]  
No error inputs will received.

> PS: Above method does not work on my CentOS 4.4 servers, and I figured out another way:

\[code\]\> uname -a  
\[/code\]  
To check your kernel's version. For me, it's 2.6.17.14\.  
Go to get the kernel's source.  
\[code\]\> wget http://kernel.org/pub/linux/kernel/v2.6/linux-2.6.17.14.tar.bz2  
\> tar zxjf linux-2.6.17.14.tar.bz2  
\> cd linux-2.6.17.14  
\> cp /boot/config-2.6.17.14.WBXsmp .config  
\> make oldconfig  
\> make prepare  
\> make modules\_prepare  
\[/code\]  
Get fuse's source code down.(I use FUSE for gluster, an open source distributted file system, it pathes fuse for its own optimization)  
\[code\]\> wget http://ftp.gluster.com/pub/gluster/glusterfs/fuse/fuse-2.7.4glfs11.tar.gz  
\> tar zxvf fuse-2.7.4glfs11  
\> cd fuse-2.7.4glfs11  
\> ./configure --with-kernelsource=/usr/src/linux-2.6.17.14  
\> ./configure && make && make install  
\[/code\]  
It should be ok. In my case, I still got "Module fuse not found." error once I reboot my system.  
I fixed that this way, may be can work for you too:  
\[code\]\> echo "mknod /dev/fuse -m 0666 c 10 229"\>\>/etc/rc.local  
\[/code\]

[0]: http://rpmrepo.org/RPMforge/Using