---
layout:     post
title:      使用jstack排查线上故障：高CPU占用
subtitle:   你公司的线上服务器CPU达到100%了么？
date:       2018-06-22             
author:     jessehzx                
header-img: img/post-bg-os-metro.jpg
catalog: 	  true
tags:
    - 高CPU占用
        
---

> 版权声明：本文为 jessehzx 原创文章，支持转载，但必须在明确位置注明出处！！！

《深入理解Java虚拟机（第2版）》看到第4章 - 【虚拟机性能监控与故障处理工具】，该章节中罗列了各种JDK的命令行工具，其中jstack令我印象深刻。还记得是去年，我们遇到了线上虚拟机CPU占用过高，导致触发监控平台一直发短信告警的故障。当时就是利用了jstack命令帮助我们跟踪Java堆栈信息，从而排查出代码问题的。

一个应用占用CPU很高，除了确实是计算密集型应用之外，通常原因都是出现了死循环。我们以当时出现的实际故障为例，来介绍怎么定位和解决这类问题。

### 排查步骤

> 思路：找出tomcat 进程中使用CPU最高、时间最长的线程，分析堆栈信息，定位问题。

1、	先拿到tomcat进程ID 

```
ps –ef | grep tomcat
```
记录下tomcat应用进程的ID: 30027（我拿到的是这个值）

> 这里按照我们排查故障的思维习惯，是直接怀疑我们部署的tomcat应用服务出现故障。如果你不确定，比如怀疑是同时部署到这台机的监控程序或其他进程引起的问题，那可以先使用top命令查看到底是哪个进程占用CPU过高，再拿这个PID走下面的流程。

2、	拿到CPU占用最高、时间最长的线程ID

```
# 显示进程号为30027的进程信息，CPU、内存占用率等
top -H -p 30027
```
![](https://ws3.sinaimg.cn/large/006tNc79gy1fsjmoa4h15j30pa08wn7z.jpg) 

> 当然这一步你也可以使用以下这个命令，显示特定PID下全部线程列表，以定位到是哪个线程占用CPU时间最高：

```
ps -mp pid -o THREAD,tid,time
``` 
  
3、	将需要的线程ID转换为16进制格式

```
printf "%x\n" 1852
```
得到结果：73c（这里我只查看了1852这个线程ID）

4、打印线程的堆栈信息

```
jstack 30027 |grep 73c -A 30
```
得到如下信息：

```
"pool-1617252-thread-1" prio=10 tid=0x00007f76a00e5800 nid=0x73c runnable [0x00007f7658e61000]
   java.lang.Thread.State: RUNNABLE
        at java.util.Random.next(Random.java:139)
        at java.util.Random.nextDouble(Random.java:394)
        at java.lang.Math.random(Math.java:695)
        at com.xxx.xxx.utils.Rand.randomNoSame(Rand.java:100)
        at com.xxx.xxx.videos.GetVideoListService.getUserPullDownVirtualPageResult(GetVideoListService.java:275)
        at com.xxx.xxx.videos.GetVideoListService.getVideoList(GetVideoListService.java:94)
        at sun.reflect.GeneratedMethodAccessor1123.invoke(Unknown Source)
```

找到出现问题的代码了！

现在来分析下具体的代码：Rand.randomNoSame(Rand.java:100)
Rand是应用封装的一个获取随机且不重复列表的工具类。randomNoSame函数的代码如下：

![](https://ws4.sinaimg.cn/large/006tNc79gy1fsjlk401rzj30x80t2dl5.jpg)

问题就出在标红的代码部分。如果第二次生成的随机数和第一次生成的随机数相等，那flag为false，同时break;只是跳出了当前的for循环，而while循环将一直进行下去。问题的根源找到了，至于具体怎么修改就看业务逻辑和需求了。

 
### 总结

最后，总结下排查CPU故障的方法和技巧有哪些：

1. top命令：Linux命令。可以查看实时的CPU使用情况。也可以查看最近一段时间的CPU使用情况。
2. PS命令：Linux命令。强大的进程状态监控命令。可以查看进程以及进程中线程的当前CPU使用情况。属于当前状态的采样数据。
3. jstack：Java提供的命令。可以查看某个进程的当前线程栈运行情况。根据这个命令的输出可以定位某个进程的所有线程的当前运行状态、运行代码，以及是否死锁等等。

<p align="center">（EOF）</p>

<p align="center"><img src="https://ws3.sinaimg.cn/large/006tKfTcgy1fs2fjgw2icj30im0lk0um.jpg" width="300" height="348"/></p>