---
layout: post
title: 找出c/cpp项目中无用的头文件
date: 2012-09-18 15:20:35.000000000 +08:00
categories:
- 技术
tags:
- cpp
- shell
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  dsq_thread_id: '1536857826'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
需要说明的是，题目可能有点歧义。这篇文章并不提供如何找代码中多余的\#include语句，而是找出一堆.h里没有被现有代码\#include到的文件。

## 背景

新的项目依赖另外一个team手上的一个老项目。虽说老项目代码已经足够稳定，但由于对方team人员的流失和文档的极度不完善，在新项目的实施过程中很多问题还都需要自己像hack一样地去解决。

其中的一个问题是，老项目直接丢给我们一个包含200多个.h的文件夹。而我们用了哪些个到最后快交付时也没人搞得清楚。

作为一个有代码洁癖的强迫症患者,这事儿必须给弄了。

## 需求

所以需求是这样的：

> 我们自己的代码有一堆.cpp .h，其中include了一堆老项目的.h， 这个比较明确，也容易处理。我们把这种include叫做直接include。  
> 但是老项目的.h间也互相include着，被直接include着的.h包含的.h，我们叫他们为间接include
> 
> 所以我们就是要把所有直接include和间接include都找出来，除此之外的从我们自己的项目里都干掉。

其实类似的事情可以通过一些静态代码工具比如pclint做到，但这些东西要么收费，要么安装使用麻烦，而且对于我的这个需求来说有点过分杀鸡牛刀了。google了几下，没找到现成的，于是决定自己写脚本解决。

## 实现

下面的这个脚本接受2个参数：1是我们项目的根文件，一般就是main所在的cpp； 2是脚本的输出文件，包含我们项目所有直接、间接引用的.h

\[bash\]  
if \[ $\# -lt '2' \]; then  
echo "rootfile outfilename"  
exit 1  
fi

rootfile=$1  
filelist=$2

if \[ ! -f $rootfile \] ;then  
echo "should be a file"  
exit 1;  
fi  
echo $rootfile $filelist

if \[ ! -z "$3" \] ;then  
echo "will delete old generated files"  
if \[ -f $filelist \]; then  
rm $filelist  
rm $filelist.top -f  
fi  
touch $filelist  
fi

includes=\`grep "\#include" $1 | awk '{print $2}' | grep -v \\< | grep \\\\.h | awk -F\\" '{print $2}'\`  
for include in $includes  
do

hfiles=\`find src -name "$(basename $include)"\`  
for hfile in $hfiles  
do  
if \[ -f $hfile \]; then  
\# echo "check $hfile..."  
basefile=\`basename $hfile\`  
if \[ -z "$(cat $filelist | grep $basefile)" \]; then  
basename $hfile \>\> $filelist  
echo "\["$rootfile","$hfile"\]" \>\> $filelist.top  
$0 $hfile $filelist  
fi  
fi

done  
done

\[/bash\]

细心的你一定注意到了，这其实是一个递归调用的脚本，不断地调用自己找到目标文件\#include的头文件。而输出文件除了最终输出的作用，还在执行期间起了一个去重的作用，以避免重复记录。  
最终的输出还有一个.top文件用于记录include的依赖关系，你甚至可以用这个文件来画一个依赖图。

有了这个脚本，输入我们的rootfile，和最终输出文件filelist。执行后得到了：

\[bash\]  
\> wc -l filelist  
77 filelist  
\[/bash\]

可见我们的项目一共有77个直接、间接依赖。  
有了这个列表，就可以只保留那200多个h里的77个，其他都可以rm掉。

舒服了。