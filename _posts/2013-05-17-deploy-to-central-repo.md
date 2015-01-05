---
layout: post
title: Depoly your own maven lib/plugin to Maven Central Repository
date: 2013-05-17 15:24:09.000000000 +08:00
categories:
- 技术
tags:
- Maven
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  _syntaxhighlighter_encoded: '1'
  dsq_thread_id: '1532655022'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
最近提交了[自己写的一个maven插件][0]到官方Maven Central Repository：[http://search.maven.org/\#browse%7C1637367917][1]，前后花了三四天时间。花这么多天并不是说这个事情有多麻烦，只是期间不得不有停顿，你没办法一气儿做好。

## 起因

因为是一个挺简单的插件，所以其实最开始并没想着搞这么复杂。参照着[http://stackoverflow.com/questions/14013644/hosting-a-maven-repository-on-github][2], 在GitHub上Host了一个[repository][3]。任何一个想用这个plugin的人只需要在自己的pom.xml里加上一个pluginRepository就可以了：

\[xml\]  
<pluginRepositories\>  
<pluginRepository\>  
<id\>swagger-maven-plugin-mvn-repo</id\>  
<url\>https://github.com/kongchen/swagger-maven-plugin/raw/mvn-repo/</url\>  
<snapshots\>  
<enabled\>true</enabled\>  
<updatePolicy\>always</updatePolicy\>  
</snapshots\>  
</pluginRepository\>  
</pluginRepositories\>  
\[/xml\]

项目放上去后也就没再管过，直到有天突然发现一大堆人抱怨说用的时候maven找不到插件，折腾了一圈发现原来是自己不慎，在写README.md时把**pluginRepository**写成了**repository**。

在issue的抱怨里有人说你不如搞一个放在Maven Central Repo里，免得让人有额外的配置。  
想想这么多人抱怨，至少说明还有人想用，就搞一下吧。

## 过程

其实参照

[https://docs.sonatype.org/display/Repository/Sonatype+OSS+Maven+Repository+Usage+Guide][4]

参照这个guide，任何人都能很方便的做到这个事情。这里简单再复述一下，顺带提示一些我遇到的小陷阱。

### 准备

#### 这里**推荐本地开发环境使用Mac或者Linux**。

因为在GPG, git这些tool在处理windows路径上做的不够好，会有些莫名其妙的bug。比如git push的时候莫名其妙的卡住，gpg说public key文件找不到等等。在这些事情上浪费时间实在划不来。

下面这些tool也是必须的:

* JDK 5+
* Maven 2.2.1+
* git/svn命令行 这取决于你的SCM是什么，我这个工程是基于git的，所以就不需要svn。
* GPG client
  * **如果你真的想让自己开发的包同步到Central Repo，这一步非常重要**
  * 参照[https://docs.sonatype.org/display/Repository/How+To+Generate+PGP+Signatures+With+Maven][5]
    * 安装好gpg client
    * 生成好key
    * upload public key到hkp://pool.sks-keyservers.net/

**注册Sonatype OSSRH (OSS Repository Hosting Service)账号**

我们八成的工作需要借助Sonatype提供给我们的平台来完成，所以注册账号是必须的。

**在OSSRH提交一个JIRA ticket，表示你打算创建一个Repository**

这一步问题不大，需要的只是时间等人来帮你处理。按照Guide里的把该填的都填对，很快就会有人给你解决。

开头说我做这个事情花了三四天时间，就是因为这一步让我有了停顿，人家那边JIRA其实早给我处理了，但我隔了两天才开始接着搞下面的事情：

### 开始

首先要把你的pom完善起来，需要填写的有：

* <modelVersion\>
* <groupId\>
* <artifactId\>
* <version\>
* <packaging\>
* <name\>
* <description\>
* <url\>
* <licenses\>
* <scm\><url\>
* <scm\><connection\>
* <developers\>

上面这些是Guide里列出来的，实际使用时我发现还得正确配置<scm\><developerConnection\> 否则maven的release插件会在relase:prepare阶段出错，原因是它默认选择svn去check in它改动过的pom.xml而全然不顾我们的项目是git的。

因为Guide里的配置是svn，而我是git。这里share一下我的scm部分的配置：

\[xml\]  
<scm\>  
<connection\>scm:git:git@github.com:kongchen/swagger-maven-plugin.git</connection\>  
<url\>scm:git:git@github.com:kongchen/swagger-maven-plugin.git</url\>  
<developerConnection\>scm:git:git@github.com:kongchen/swagger-maven-plugin.git</developerConnection\>  
</scm\>  
\[/xml\]

接下来就是按照[Guide里章节7a][6]去做了。

如果你只想部署到Sonatype的snapshot repo，参照7a.1和7a.2就可以了。但如果想同步到Central Repo，就需要参照7a.3 Stage a Release先Stage一个Release出来。

**Stage a Release说白了就是三步**

\[bash\]  
$ mvn release:clean  
$ mvn release:prepare  
$ mvn release:perform  
\[/bash\]

但这三步想一次成功也不太容易。

* 首先必须保证本地代码库跟SCM上up to date。
* 其次，本地的pom里的version必须是个SNAPSHOT。

假设你希望Central Repo里面发布1.1版本，那此时此刻你pom.xml里必须是1.1-SNAPSHOT。  
而mvn release:prepare会做这样2件事情：

    1. 把你pom.xml里的version中的SNAPSHOT去掉，比如原来是1.1-SNAPSHOT，它会给改成1.1
    2. 将改好的pom.xml提交到SCM。针对于git，就是commit + push.

* 最后的mvn relase:perform则是基于prepare的结果，把相关文件提交到Sonatype的release仓库。这里就会用到你的GPG key。

三步中任何一步中的任一个步骤出错都需要**重来**。比较折腾人的是如果pom.xml里version已经被mvn release:prepare从1.1-SNAPSHOT改成了1.1并push上了git server。**重来**就意味着得把这里再改回来，不然prepare过不了。你光改本地的还不行，因为prepare会抱怨你本地的跟git server上的不一致，也过不了。

**这三步成功之后，事情还没有结束，还有三步需要做。**

1. 登陆[Nexus UI][7], 找到刚刚stage成功的包  
[![](assets/staging_close-300x196.png)][8]
2. Close这个repo。如果之前的操作都没问题，close才会成功。否则你会看到一些失败，通常是pom里有些东西配置错误，或者GPG key有问题等等。
3. Close成功后，就可以点击Release按钮Release了。在此之前，Release按钮都是灰化的。

**这三步完成之后，就真的只剩最后一步了。**

### 搞定

回到你最开始创建的JIRA ticket里，加一个comment说你已经release了，希望能够被sync到Central Repo。

之后如果没什么意外的话，你的库就会出现在Maven Central Repository了。

这个事情只需做一次，以后包的更新只需要release成功，Sonatype会每两小时向Central Repo同步一次。

[0]: https://github.com/kongchen/swagger-maven-plugin
[1]: http://search.maven.org/#browse%7C1637367917
[2]: http://stackoverflow.com/questions/14013644/hosting-a-maven-repository-on-github
[3]: https://github.com/kongchen/swagger-maven-plugin/tree/mvn-repo
[4]: https://docs.sonatype.org/display/Repository/Sonatype+OSS+Maven+Repository+Usage+Guide
[5]: https://docs.sonatype.org/display/Repository/How+To+Generate+PGP+Signatures+With+Maven
[6]: https://docs.sonatype.org/display/Repository/Sonatype+OSS+Maven+Repository+Usage+Guide#SonatypeOSSMavenRepositoryUsageGuide-7a.DeploySnapshotsandStageReleaseswithMaven
[7]: https://oss.sonatype.org/
[8]: http://www.kongch.com/wp-content/uploads/2013/05/staging_close.png