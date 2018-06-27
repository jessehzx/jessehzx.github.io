---
layout:     post
title:      使用jmap排查线上故障：高内存占用
subtitle:   你公司的线上服务器内存爆了么？
date:       2018-06-24            
author:     jessehzx                
header-img: img/post-bg-os-metro.jpg
catalog: 	  true
tags:
    - 高内存占用
        
---

> 版权声明：本文为 jessehzx 原创文章，支持转载，但必须在明确位置注明出处！！！

### 引言
前一篇介绍了 [使用jstack排查线上故障：高CPU占用](https://mp.weixin.qq.com/s?__biz=MzU5MDYxOTc2NQ==&mid=2247483677&idx=1&sn=3bdbd1b7ce5a195a7d42016fcc2e54a7&chksm=fe3a3407c94dbd11b441c824c1d03aff9d7cb74ff21066f8b9ab90460a100d34d13d2d1a6e0b#rd)，那这一篇文章主要分析高内存占用故障的排查。

搞Java开发的，经常会碰到下面两种异常：

- java.lang.OutOfMemoryError: PermGen space

- java.lang.OutOfMemoryError: Java heap space

要详细解释这两种异常，需要简单重提下Java内存模型。

### 复习Java内存模型

Java内存模型是描述Java程序中各变量（实例域、静态域和数组元素）之间的关系，以及在实际计算机系统中将变量存储到内存和从内存取出变量这样的低层细节。

在Java虚拟机中，内存分为三个代：新生代（New）、老年代（Old）、永久代（Perm）。

1. 新生代：新建的对象都存放这里。

2. 老年代：存放从新生代New中迁移过来的生命周期较久的对象。新生代New和老生代Old共同组成了堆内存。

3. 永久代：是非堆内存的组成部分。主要存放加载的Class类级对象如class本身，method，field等等。

### 两种异常的原因

如果出现java.lang.OutOfMemoryError: Java heap space异常，说明Java虚拟机的堆内存不够。原因有二：

- Java虚拟机的堆内存设置不够，可以通过参数-Xms、-Xmx来调整。

- 代码中创建了大量大对象，并且长时间不能被垃圾收集器收集（存在被引用）。

如果出现java.lang.OutOfMemoryError: PermGen space，说明是Java虚拟机对永久代Perm内存设置不够。

一般出现这种情况，都是程序启动需要加载大量的第三方jar包。例如：在一个Tomcat下部署了太多的应用。 

从代码的角度，软件开发人员主要关注java.lang.OutOfMemoryError: Java heap space异常，减少不必要的对象创建，同时避免内存泄漏。

### 案例分析

现在以一个实际的例子分析内存占用的故障排查。

![](https://ws4.sinaimg.cn/large/006tKfTcgy1fsmn9wox3sj30ev06c400.jpg)

通过top命令，发现PID为9004的Java进程一直占用比较高的内存不释放（24.7%），出现高内存占用的故障。

想起上一篇 [使用jstack排查线上故障：高CPU占用](https://mp.weixin.qq.com/s?__biz=MzU5MDYxOTc2NQ==&mid=2247483677&idx=1&sn=3bdbd1b7ce5a195a7d42016fcc2e54a7&chksm=fe3a3407c94dbd11b441c824c1d03aff9d7cb74ff21066f8b9ab90460a100d34d13d2d1a6e0b#rd) 介绍的PS命令，能否用它来找出具体是哪个线程呢？

```
ps -mp 9004 -o THREAD,tid,time,rss,size,%mem
```

![](https://ws1.sinaimg.cn/large/006tKfTcgy1fsmn8c1helj30ez063dgv.jpg)

遗憾的是，发现PS命令可以查到具体进程的CPU占用情况，但是不能查到一个进程下具体线程的内存占用情况。

只好寻求他法，幸好Java提供了一个很好的内存监控工具：jmap命令

jmap命令有下面几种常用的用法：

- jmap [pid]

- jmap -histo:live [pid] >a.log

- jmap -dump:live,format=b,file=xxx.xxx [pid]

用得最多是后面两个。其中，jmap -histo:live [pid] 可以查看当前Java进程创建的活跃对象数目和占用内存大小。

jmap -dump:live,format=b,file=xxx.xxx [pid] 则可以将当前Java进程的内存占用情况导出来，方便用专门的内存分析工具（例如：MAT）来分析。

> MAT是Eclipse Memory Analysis Tool的缩写，是一个分析Java内存泄露的工具，推荐使用。

等不及了，下面我们使用jmap -histo:live [pid] 命令把高内存占用的妖孽揪出来：

```
# 其中head命令用于显示前100行
jmap -histo:live 2809 | head -n 100
```

![](https://ws1.sinaimg.cn/large/006tKfTcgy1fsmn7rnjqxj30de0akq4h.jpg)

从上图可以看出，int数组、constMethodKlass、methodKlass、constantPoolKlass都占用了大量的内存。

特别是占用了大量内存的int数组，务必要仔细检查相关代码，才能让内存高使用率降下来。

好了，是不是很简单，这次的故障排查就这样被我们解决了。
 
### 总结 

最后，总结下排查内存故障的方法和技巧有哪些：

1. top命令：Linux命令。可以查看实时的内存使用情况。  

2. jmap -histo:live [pid]，然后分析具体的对象数目和占用内存大小，从而定位代码。

3. jmap -dump:live,format=b,file=xxx.xxx [pid]，然后利用MAT工具分析是否存在内存泄漏等等。

我希望每一个搞计算机的人在平时都多提升自己，不要觉得这个东西用不到就不去学，而是应该抱着这个知识点很重要，我要计划好时间去学的心态，才能走得长远。

<p align="center">（EOF）</p>

<p align="center"><img src="https://ws3.sinaimg.cn/large/006tKfTcgy1fs2fjgw2icj30im0lk0um.jpg" width="300" height="348"/></p>