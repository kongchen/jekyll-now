---
layout: post
title: EPOLL下的accept
date: 2011-01-25 02:00:57.000000000 +08:00
categories:
- 技术
tags:
- epoll
- Linux
- socket
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  dsq_thread_id: '1532652538'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
不知道是谁第一个犯了错，在网上贴出所谓epoll通用框架的代码。注意看accpet的处理：  
\[c highlight="11,12,32,33"\]  
epfd = epoll\_create(10);

struct sockaddr\_in clientaddr;  
struct sockaddr\_in serveraddr;  
listenfd = socket(AF\_INET, SOCK\_STREAM, 0);

bool bReuseaddr = 1;  
//setsockopt(listenfd,SOL\_SOCKET,SO\_REUSEADDR,(const char\*)&bReuseaddr,sizeof(bool));  
setnonblocking(listenfd);  
ev.data.fd = listenfd;  
ev.events = EPOLLIN | EPOLLET;  
// ev.events=EPOLLIN;  
epoll\_ctl(epfd, EPOLL\_CTL\_ADD, listenfd, &ev);  
bzero(&serveraddr, sizeof(serveraddr));  
serveraddr.sin\_family = AF\_INET;  
// char \*local\_addr=INADDR\_ANY;  
// inet\_aton(local\_addr,&(serveraddr.sin\_addr));//htons(SERV\_PORT);  
serveraddr.sin\_addr.s\_addr = htonl(INADDR\_ANY);  
serveraddr.sin\_port = htons(SERV\_PORT);  
bind(listenfd, (sockaddr \*) &serveraddr, sizeof(serveraddr));  
listen(listenfd, LISTENQ);  
maxi = 0;  
int nfds\_count = 0, recvcount = 0, recvlen = 0;  
for (;;) {  
cout << "before epoll\_wait! nfds\_count=" << nfds\_count << endl;  
nfds = epoll\_wait(epfd, events, 20, 10000000);

// if(nfds\>1) cout<<"nfds="<<nfds<<endl;  
cout << "nfds=" << nfds << endl;

for (i = 0; i < nfds; ++i) {  
if (events\[i\].data.fd == listenfd) {  
connfd = accept(listenfd, (sockaddr \*) &clientaddr, &clilen);  
nfds\_count += 1;  
if (connfd < 0) {  
perror("connfd<0");  
exit(1);  
}  
setnonblocking(connfd);  
char \*str = inet\_ntoa(clientaddr.sin\_addr);  
cout << "connect from " << str << "connfd=" << connfd << endl;  
ev.data.fd = connfd;  
ev.events = EPOLLIN | EPOLLET;  
//ev.events=EPOLLIN;  
//ev.events=EPOLLIN|EPOLLOUT|EPOLLET;  
epoll\_ctl(epfd, EPOLL\_CTL\_ADD, connfd, &ev);

}  
}  
}  
\[/c\]  

代码是从某处（很多地方都是这段，连注释都一样）拷过来的。熟悉epoll的人看了应该很熟系，其实就是将linsten的fd也加入到epoll中，当有新连接加入时可以epoll\_wait到，随后再用accpet处理。  
事实上这段代码是有问题的------**高并发的情况下，accept到的fd的数量跟client端的发起请求的数量并不相等。**我测了一下，100个并发（其实也不算高了）往往少几个到几十个不等。

相信很多人被这段代码误导了，因为我在遇到问题的时候搜到不少帖子，用的都是这样的代码。只是奇怪的是没见几个人说遇到我提到的这个问题。不过有人说这段代码效率不高，提出用阻塞式IO把accpet提到一个单独的线程里做。撇开这样做性能上的优劣不说，倒是能解决我遇到的问题，但这治标不治本------我要用epoll+nonblockio

下面说说这段误人子弟的代码，其实 man epoll 就能找到这段代码的出处。相信它是某位同志从里面帖出来然后玩弄了许久后好心放到网上的，不然不会有这么多的//注释。而问题就是出在注释上，注意到 man-page 里的代码是这样的：  
\[c highlight="1"\]  
ev.events = EPOLLIN;  
ev.data.fd = listen\_sock;  
if (epoll\_ctl(epollfd, EPOLL\_CTL\_ADD, listen\_sock, &ev) == -1) {  
perror("epoll\_ctl: listen\_sock");  
exit(EXIT\_FAILURE);  
}  
\[/c\]

没错，man里并没有将listenfd以ET的方式加入epoll(不指定EPOLLET默认就是Level Triggered)，事实也证明不设ET就没有问题了。

难道说ET模式下就不能以非阻塞模式来accpet？答案是否定的。其实只要把accpet方式改一下就可以了，简单说就是if改while：  
\[c highlight="1"\]  
while ((confd = accept(listenfd, (struct sockaddr\*) sa, &clientlen)) \> 0) {

//add connection to epoll  
peer = (struct sockaddr\*) sa;  
printf("%s:%d connect at %d.\\n", inet\_ntoa(peer-\>sin\_addr),  
peer-\>sin\_port, confd);  
setnonblocking(confd);

ev.data.fd = confd;  
ev.events = EPOLLIN | EPOLLET ;

epoll\_ctl(myfd, EPOLL\_CTL\_ADD, confd, &ev);

}  
\[/c\]

今天有点晚，问题的根源过两天再写篇博细说...