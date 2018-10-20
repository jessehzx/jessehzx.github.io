---
layout:     post
title:      浅析Java异常
subtitle:   受检异常？非受检异常？自定义异常？
date:       2018-09-10            
author:     jessehzx                
header-img: img/post-bg-os-metro.jpg
catalog: 	  true
tags:
    - Exception
        
---

> 版权声明：本文为 jessehzx 原创文章，支持转载，但必须在明确位置注明出处！！！

### 引子
先一起来看看下面的代码：

```
package com.jessehzx.Exception;
/**
 * @author huangzx
 * @date 2018/9/10
 */
public class ExceptionTypeTest {
    public class ExceptionTypeTest {
    public int doSomething() throws ArithmeticException {
        return 5 / 0;
    }
    public static void main(String[] args) {
        ExceptionTypeTest ett = new ExceptionTypeTest();
        ett.doSomething();
    }
}
```

问：上面的代码能编译通过吗？

答：可以。ArithmeticException是RuntimeException异常，可以不进行捕获或抛出。
如果改为IOException，如下：

```
package com.jessehzx.Exception;
import java.io.*;
/**
 * @author huangzx
 * @date 2018/9/10
 */
public class ExceptionTypeTest {
    public void doSomething() throws IOException {
        InputStream is = new FileInputStream(new File("/Users/huangzx/test.txt"));
        is.read();
    }
    public static void main(String[] args) {
        ExceptionTypeTest ett = new ExceptionTypeTest();
        ett.doSomething();
    }
}
```

问：它还能编译通过吗？

答：不能。IOException是直接继承自Exception的异常，它必须得到处理，要么捕获要么抛出。

### 受检异常和非受检异常
上面两个例子中，ArithmeticException与IOException都是来自Exception体系，为什么一个不需要处理，一个需要处理呢？这就涉及到两个概念：受检异常和非受检异常。
先来复习一下Java异常体系：

![Java异常体系](https://ws2.sinaimg.cn/large/006tNbRwgy1fv4hhrmr2sj30uw0cqdgt.jpg)

- 非受检异常：RuntimeException及其子类。这类异常由程序员逻辑错误导致，应该人为承担责任。Java编译器不要求强制处理，即可以捕获或抛出，也可以不捕获或抛出。
- 受检异常：非RuntimeException异常。这类异常是由于外部的一些偶然因素引起的。Java编译器要求强制处理，即必须得到捕获或抛出。

两者的代表人物出场如下：

- 非受检异常：RuntimeException,ArithmeticException,NullPointerException,ClassCastException,ArrayIndexsOutOfBoundsException
- 受检异常：Exception,IOException,SQLException,FileNotFoundException

### 如何处理异常
既然两者的概念清晰了，处理方式也随之昭然若揭。

针对受检异常：

- throws抛出。（这是低端做法）
- try/catch捕获处理。（推荐之，高端做法）

针对非受检异常：

- throws抛出。
- try/catch捕获处理。
- 不处理。（这一波可以，我才不管程序崩没崩，把锅甩给上帝）

### 自定义异常
以上是打基础，下面来玩点高级的。

> 当需要一些跟特定业务相关的异常信息类时，我们可以在Java自身定义的异常之外，编写继承自Exception的受检异常类，也可以编写继承自RuntimeException或其子类的非受检异常类。

一般情况下，异常类提供了默认构造器和一个带有String类型参数的构造器。我们的自定义异常，只需要实现这两个构造器就足够用了。因为最有价值的是我们定义的异常类型（即异常的名字），当异常发生时，我们只要看到了这个名字就知道发生了什么。所以除非有一些特殊操作，否则自定义异常只需简单实现构造器即可。

示例：

1、自定义业务异常类，继承自RuntimeException。

```
package com.jessehzx.Exception;

/**
 * @author huangzx
 * @date 2018/9/10
 */
public class BusinessException extends RuntimeException {
    private String code;    // 异常返回码
    private String msg;     // 异常信息
    public BusinessException() {
        super();
    }
    public BusinessException(String message) {
        super(message);
        this.code = code;
    }
    public BusinessException(String code, String msg) {
        super();
        this.code = code;
        this.msg = msg;
    }
    public String getCode() {
        return code;
    }
    public String getMsg() {
        return msg;
    }
}
```
2、定义方法，声明throws这个自定义异常。要使用它，必须通知调用代码的类，做好抱接这个异常的心理准备。代码逻辑中异常发生的点，需要throw抛出。

```
public void logicCode() throws BusinessException {
    throw new BusinessException("-1000", "业务出错");
}
```
3、测试自定义异常main方法

```
public static void main(String[] args) {
    ExceptionTest et = new ExceptionTest();
    try {
        et.logicCode();
    } catch (BusinessException e) {
        e.printStackTrace();
        System.out.println("code=" + e.getCode() + "msg=" + e.getMsg());
    }
}
```
基本上，就差不多了。另外还有一个需要注意的点，就是：千万不要在finally再抛出一个非常规异常，因为它必定会执行，导致你try{}监测的代码块捕获的异常得不到处理，会起到混淆视听的效果，让你排查问题时摸不着头脑，请切记。示例如下：

```
package com.jessehzx.Exception;

/**
 * @author huangzx
 * @date 2018/9/10
 */
public class ExceptionLoseTest {
    public static int throwException() throws Exception {
        try {
            throw new Exception();
        } catch (Exception e) {
            throw e;
        } finally {
            throw new NullPointerException();
        }
    }

    public static void main(String[] args) {
        try {
            throwException();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
执行结果如下：

```
java.lang.NullPointerException
	at com.jessehzx.Exception.ExceptionLoseTest.throwException(ExceptionLoseTest.java:14)
	at com.jessehzx.Exception.ExceptionLoseTest.main(ExceptionLoseTest.java:20)
```

### 总结
- 简单理解受检异常和非受检异常：非受检异常就是RuntimeException及其子类的异常，其余的就理解为受检异常
- 自定义异常是为了特定业务需求，让你通过异常类型来辨别发生了什么异常，以便更好的修正和优化你的逻辑代码用的
- try/catch那些你已经知道如何处理的异常，throw那些你还不知怎么处理的异常。
