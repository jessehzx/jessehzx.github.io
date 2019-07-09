---
layout:     post
title:      [面试]Thread 中 start() 和 run() 的区别都不知道，还怎么混？
subtitle:   聊一聊 Java 线程中这两个方法不一样的点 ...
date:       2019-07-08             
author:     jessehzx                
header-img: img/post-bg-building.jpg
catalog: 	  true
tags:
    - 程序人生
        
---

> 微信公众号：JesseTalkJava

### 引子

最近面试了不少 Java 工程师，有一些心得体会想给大家分享，比如，上次就有一个小哥被我 “送走” 了，我尽量复原一下当时的面试情景，对话大致如下：

我：我们知道，JDK 中的线程 Thread 类有两个方法，一个叫 start()，一个叫 run()，那你知道两者有什么区别吗？

小哥：额......好像都是运行...运行一个线程吧......

我：那都是运行一个线程的话，设计成一个方法就好了，没必要两个方法吧？

小哥：这......不太清楚......

我：好，今天就这样吧...（起身）

【PS：当然，最后这个起身动作是前面已经问了好几个问题之后产生的。】

### 分析

众所周知，Java 面试一般都逃不开多线程的题目。我一般会从简单的开始问，如果答得上来就继续追问，几番深入浅出之后对求职者的掌握程度就有一个基本的判定了。

如果求职者连最简单的问题都答不上来，那我就会觉得要么基础太差，要么对此次面试没有给予足够的重视去作准备。不管是何种原因，在面试官心中都是掉分的，结果也可想而知。

### 原理

好，回过神来，再敲一下黑板：Thread 类中的 start() 和 run() 到底有什么不一样？

在正式学习 Thread 类中的这两方法之前，我们先来了解一下线程有哪些状态，这个将会有助于后面对 Thread 类中的方法的理解。

- 创建（new）状态: 准备好了一个多线程的对象
- 就绪（runnable）状态: 调用了 start() 方法, 等待 cpu 进行调度
- 运行（running）状态: 执行 run() 方法
- 阻塞（blocked）状态: 暂时停止执行, 可能将资源交给其它线程使用
- 终止（dead）状态: 线程销毁

实现并启动线程的两种方式，这可是我们背得滚瓜烂熟的知识点：

- 编写一个类继承自 Thread 类，重写 run() 方法。用 start() 方法启动线程
- 编写一个类实现 Runnable 接口，实现 run() 方法。用 new Thread(Runnable target).start() 方法来启动线程

为了分清 start() 方法调用和 run() 方法调用的区别，请看下面一个例子：

```java
package cn.java.basic;

/**
 * @author Jessehuang
 */
public class Test {
    public static void main(String[] args) {
        System.out.println("主线程ID:" + Thread.currentThread().getId());
        MyThread thread1 = new MyThread("thread1");
        thread1.start();
        MyThread thread2 = new MyThread("thread2");
        thread2.run();
    }
}

class MyThread extends Thread {
    private String name;

    public MyThread(String name) {
        this.name = name;
    }

    @Override
    public void run() {
        System.out.println("name:" + name + " 子线程ID:" + Thread.currentThread().getId());
    }
}

```

运行结果：

```
主线程ID:1
name:thread2 子线程ID:1
name:thread1 子线程ID:10
```

从输出结果可以得出以下结论：

1）thread1 和 thread2 的线程 ID 不同，thread2 和主线程 ID 相同，说明通过 run() 方法调用并不会创建新的线程，而是在主线程中直接运行 run() 方法，跟普通的方法调用没有任何区别。

2）虽然 thread1 的 start 方法调用在 thread2 的 run() 方法前面调用，但是先输出的是 thread2 的 run() 方法调用的相关信息，说明新线程创建的过程不会阻塞主线程的后续执行。

对于 Thread 类中 start() 和 run() 的源码，这里作一个粗浅的说明：start() 方法调用了本地 native 的 start0()，而 run() 就是直接用重写了 Thread 实现的 Runnable 的 run()。比较简单，就不贴了，可自行在 JDK 查阅。

OK，我们稍作整理成较为正式（但略显枯燥的）文字，来阐释两者的区别：

1、start() 方法

它的作用是启动一个新线程。

通过它启动的新线程，是处于就绪（可运行）状态的，此时还没有真正运行，等待 cpu 调度，一旦它得到 cpu 时间片，就会开始执行相应线程的 run() 方法了，这里 run() 方法称为线程体，它包含了要执行的这个线程的内容，run() 方法运行结束，此线程随即终止。

用 start() 方法来启动线程，真正实现了多线程运行，即无需等待某个线程的 run 方法体代码执行完毕就能够直接继续执行下面的代码，即进行线程切换。

2、run() 方法

就是一个普通的成员方法，没关系，你反复调用它都可以，反正它也不会帮你启动一个新线程。

程序中只有一个线程，即主线程，所以它的执行路径就只有一条，从上到下按顺序来执行，这就导致要等待 run 方法体执行完毕后，才可以继续执行下面的代码。也就是说没有实现多线程。

那怎么**理解『多线程』**这个概念？

举个例子，正值暑假，你家来了一波熊孩子，争着要玩你的 psp 游戏机，但你家 psp 就这么一台，为了让每个熊孩子都能玩上游戏机，你叫他们排起队，一人玩完接下一个人玩。那么 psp 就相当于 cpu，排队这个动作就是线程中的 start() 方法，等 cpu 选中一个熊孩子就轮到他，他就 run() 起来开始玩游戏，当 cpu 的运行时间片执行完，这个线程就继续排队，等待下一次的 run() 才能玩。

多线程其实就是分时利用CPU，宏观上让所有线程一起执行，殊不知，是 CPU 在做着极快速的上下文切换，给你营造出一种眼花缭乱的错觉，还真以为是同时执行的呢。

### 总结

好了，文章开头我们虽然“送走”了小哥，但作为一枚好心人，还是帮他总结回答下吧：

1. start() 可以启动一个新线程，run() 不能
2. start() 不能被重复调用，run() 可以
3. start() 中的 run 代码可以不执行完就继续执行下面的代码，进行线程切换，而调用 run() 方法必须等待其代码全部执行完才能继续执行下面的代码
4. start() 实现了多线程，run() 没有