---
layout: post
title: 做了道题
date: 2011-07-29 13:29:42.000000000 +08:00
categories:
- 技术
tags:
- algorithm
- C
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  dsq_thread_id: '1560502824'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
![](assets/ppp.jpg)  
题目大意是提供四个六面颜色各异的立方体，让你垒起来，得到的长方体除掉top和bottom的四面每一面小格子颜色都不重复。  
思路就是找出每个立方体能够被用作大长方体四面的部分，然后慢慢用穷举法拼，当然其间有剪枝。

开始犯了个算是致命的错误------------在找每个立方体能被用作大长方体四面部分的函数，也就是get\_round函数，起初只考虑了一个方向,结果输出空白。这个陷阱挺坏的......41-58行加上后，输出应该是正确的了。

\[c highlight="41-58,"\]  
\#include <stdio.h\>

\#define R 1  
\#define B 2  
\#define G 4  
\#define Y 8

\#define FRONT 0  
\#define BACK 1  
\#define LEFT 2  
\#define RIGHT 3  
\#define TOP 4  
\#define BOTTOM 5

static int CUBEA\[6\] = {R,B,G,Y,B,Y};  
static int CUBEB\[6\] = {R,G,G,Y,B,B};  
static int CUBEC\[6\] = {Y,B,R,G,Y,R};  
static int CUBED\[6\] = {Y,G,B,R,R,R};

void get\_round(int \* cube, int\* round, int side){  
switch(side){  
case 0:  
round\[0\] = cube\[FRONT\];  
round\[1\] = cube\[RIGHT\];  
round\[2\] = cube\[BACK\];  
round\[3\] = cube\[LEFT\];  
return;  
case 1:  
round\[0\] = cube\[FRONT\];  
round\[1\] = cube\[TOP\];  
round\[2\] = cube\[BACK\];  
round\[3\] = cube\[BOTTOM\];  
return;  
case 2:  
round\[0\] = cube\[TOP\];  
round\[1\] = cube\[RIGHT\];  
round\[2\] = cube\[BOTTOM\];  
round\[3\] = cube\[LEFT\];  
return;

case 3:  
round\[3\] = cube\[FRONT\];  
round\[2\] = cube\[RIGHT\];  
round\[1\] = cube\[BACK\];  
round\[0\] = cube\[LEFT\];  
return;  
case 4:  
round\[3\] = cube\[FRONT\];  
round\[2\] = cube\[TOP\];  
round\[1\] = cube\[BACK\];  
round\[0\] = cube\[BOTTOM\];  
return;;  
case 5:  
round\[3\] = cube\[TOP\];  
round\[2\] = cube\[RIGHT\];  
round\[1\] = cube\[BOTTOM\];  
round\[0\] = cube\[LEFT\];  
return;  
}

}  
char\* to\_color(int i){  
switch(i){  
case 1: return "R";  
case 2: return "B";  
case 4: return "G";  
case 8: return "Y";  
default: printf("errorinput:%d\\n",i); return "ERR";  
}  
}  
void print\_result(int\* cube, int side, int off){  
int r\[4\]={0};  
get\_round(cube,r,side);  
printf("%s,%s,%s,%s", to\_color(r\[(0 + off) % 4\]), to\_color(r\[(1 + off) % 4\]), to\_color(r\[(2 + off) % 4\]), to\_color(r\[(3 + off) % 4\]));  
}

void check\_rounds(int as,int\* a,int bs, int\* b, int cs, int\* c, int ds,int\*d)  
{  
int has = 1;  
for(int ai=0;ai<4;ai++){  
for(int bi=0;bi<4;bi++){  
if(a\[ai\]!=b\[bi\])  
for(int ci=0;ci<4;ci++){  
if(a\[ai\]!=c\[ci\] && b\[bi\] != c\[ci\])  
for(int di=0;di<4;di++){  
has = 1;  
for(int i=0;i<4;i++){  
if((a\[(ai + i) % 4\]|b\[(bi + i) % 4\]|c\[(ci + i) % 4\]|d\[(di + i) % 4\])!=15) {  
has = 0; break;  
}  
}

if(has == 1){  
printf("\\t1-");  
print\_result(CUBEA,as,ai);  
printf("\\n\\t2-");  
print\_result(CUBEB,bs,bi);  
printf("\\n\\t3-");  
print\_result(CUBEC,cs,ci);  
printf("\\n\\t4-");  
print\_result(CUBED,ds,di);  
printf("\\n\\t---------\\n");  
return;  
}  
}  
}  
}  
}  
}

int main(int argc, char\*\* argv){  
int rounda\[4\] = {0};  
int roundb\[4\] = {0};  
int roundc\[4\] = {0};  
int roundd\[4\] = {0};

for(int i=0;i<6;i++){  
get\_round(CUBEA,rounda,i);  
for(int ii = 0;ii<6;ii++){  
get\_round(CUBEB,roundb,ii);  
for(int iii=0;iii<6;iii++){  
get\_round(CUBEC,roundc,iii);  
for(int iiii=0;iiii<6;iiii++){  
get\_round(CUBED, roundd, iiii);  
check\_rounds(i, rounda, ii, roundb, iii, roundc, iiii, roundd);  
}  
}  
}  
}  
return 0;

}  
\[/c\]

结果如下：

> 1-R,B,B,Y  
> 2-G,Y,R,G  
> 3-B,G,Y,R  
> 4-Y,R,G,B  
> ---------  
> 1-Y,B,B,R  
> 2-G,R,Y,G  
> 3-R,Y,G,B  
> 4-B,G,R,Y  
> ---------
>