---
layout:     post
title:      理解Spring AOP其实并不难
subtitle:   讲述Spring面向切面编程并通过代码实例去理解它
date:       2018-05-04             
author:     jessehzx                
header-img: img/post-bg-os-metro.jpg
catalog: 	  true
tags:
    - Spring AOP
        
---

> 版权声明：本文为 jessehzx 原创文章，支持转载，但必须在明确位置注明出处！！！

### 引子

我们开发人员都有一种恐惧就是，当需求有增加时，需要在原来的代码上修改，这得多麻烦而且费时，还可能会将原来的代码改出bug，还有就是需要在方法调用之间加上日志记录，要在那么多个方法同时加上，代码的重复率很高，那么有没有什么方法可以满足以上两个很常见的问题呢？

Spring的AOP(Aspect Oriented Programming)面向切面编程可以完美解决。面向切面编程指的是在原来代码的基础上，加上增强的部分，生成一个代理的对象，也就是说，程序员可以在不修改原代码的情况下，实现对目标类的增强，其原理就是将一个被AOP动态增强的类，通过JDK动态代理或CGLib动态代理模式生成一个代理类。

### AOP的一些相关概念

- 切面（Aspect）：是切入点和增强的结合；
- 连接点（Joinpoint）：类里面可以被增强的方法，这些方法称为连接点；
- 增强处理（Advice）：指拦截到Joinpoint之后所要做的事情就是增强。增强分为前置增强，后置增强，环绕增强，异常抛出增强，最终增强；
- 切入点（Pointcut）：指我们要对哪些Joinpoint进行拦截的定义。

### AspectJ语法

```
"execution(* top.jessehzx.controller.v1.*.*(..))"
```

- 指定在执行 top.jessehzx.controller.v1 包中任意类的任意方法之前执行方法增强；
- 第一个星号表示返回值不限；
- 第二个星号表示类名不限；
- 第三个星号表示方法名不限；
- 圆括号中的 .. 表示任意个数、类型不限的形参。

### AOP增强处理方法

**Before**

```
@Aspect
public class LogAspect {
  @Before("execution(* top.jessehzx.test.*.*(..))")
  publc void doBefore() {
    System.out.println("test before");
  }
}
```

用于目标方法被调用前做一些增强处理。

**AfterReturning**

```
@Aspect
public class LogAspect {
  @AfterReturning(returning="rvt", pointcut="execution(* top.jessehzx.test.*.*(..))")
  publc void doAfterReturning(Object rvt) {
    System.out.println("获取返回值：" + rvt);
  }
}
```

用于访问目标方法返回值，并作相关处理。

**After**

```
@Aspect
public class LogAspect {
  @After("execution(* top.jessehzx.test.*.*(..))")
  publc void doAfter() {
    System.out.println("test after");
  }
}
```

用于目标方法被调用后做一些增强处理。与AfterReturning有些相似，但是也有区别，AfterReturning只有在目标方法成功完成后才会被织入。

**Around**

```
@Aspect
public class LogAspect {
  @Around("execution(* top.jessehzx.test.*.*(..))")
  publc void doAround(ProceedingJoinPoint pjp) {
    System.out.println("around before");
    pjp.proceed();
    System.out.println("around after");
  }
}
```

ProceedingJoinPoint 参数是必须的，因为要让目标方法调用，那么必须调用其方法proceed()。

**AfterThrowing**

```
@Aspect
public class LogAspect {
  @AfterThrowing(throwing="ex", pointcut="execution(* top.jessehzx.test.*.*(..))")
  publc void doThrowing(Throwable ex) {
    System.out.println("目标方法抛出异常：" + ex);
  }
}
```
用于处理目标方法抛出的异常。

### 实战

**定时任务中加入日志打印**

定义一个切面：

```
@Aspect
public class LogInterceptor {

  // 定义一个切入点
  @Pointcut("execution(* top.jessehzx.handler.*(..))")
  private void log() {}

  // before增强
  @Before("log() && args(params)")
  private void before(String params) {
    XxlJobLogger.log("[定时任务] " + params + " >>>>>>>>>>>>>>>>>>> 开始");
  }

  // After增强
  @After("log() && args(params)")
  private void after(String params) {
    XxlJobLogger.log("[定时任务] " + params + " >>>>>>>>>>>>>>>>>>> 结束");
  }
}
```

目标方法：

```
package top.jessehzx.handler.topic;

import top.jessehzx.remote.TopicRemoteService;
import com.xxl.job.core.biz.model.ReturnT;
import com.xxl.job.core.handler.IJobHandler;
import com.xxl.job.core.handler.annotation.JobHander;
import com.xxl.job.core.log.XxlJobLogger;
import org.springframework.stereotype.Service;

import javax.annotation.Resource;

/**
 * @author jessehzx
 * @date 2018/5/4
 */

@JobHander(value="pushAuthHandler")
@Service
public class PushAuthHandler extends IJobHandler {

    @Resource
    private TopicRemoteService topicRemoteService;

    @Override
    public ReturnT<String> execute(String... params) throws Exception {
        topicRemoteService.pushAuthMessage();
        return ReturnT.SUCCESS;
    }
}
```

**定时任务中加入方法锁**

定义一个切面：

```
@Aspect
public class TaskLockAspect {

  // 此处省略部分代码

  @Pointcut("@annotation(top.jessehzx.annotation.LockMethod)")
  public void pointCut() {
  }

  @Around("pointCut()")
  public Object doAround(ProceedingJoinPoint pjp) {
    String targetName = pjp.getTarget().getClass().getName();
    String methodName = pjp.getSignature().getName();
    String lockKey = targetName + "#" + methodName;
    if(!publicLock.getLock(lockKey)) {
      Long time = Long.valueOf(valueRedisTemplate.get(lockKey));
      logger.info("任务已被锁定lockKey:{}, 锁定时间:{}", lockKey, DateUtil.format(new Date(time), "yyyy-MM-dd HH:mm:ss"));
      return null;
    }

    try {
      Object result = pjp.proceed();
      publicLock.releaseLock(lockKey);
      return result;
    } catch (Throwable throwable) {
      throwable.printStackTrace();
      publicLock.releaseLock(lockKey);
    }
    return null;
  }
}
```

目标方法：

```
package top.jessehzx.controller.v1.remote;
// 此处省略部分代码

@RestController
@RequestMapping("/remote/topic")
public class TopicRemoteController {

    private final Logger logger = LoggerFactory.getLogger(TopicRemoteController.class);

    @Resource
    private TopicAuthService topicAuthService;

    @LockMethod
    @RequestMapping(method = RequestMethod.GET, value = "task/pushAuthMessage")
    public void pushAuthMessage() {
        topicAuthService.pushAuthMessage();
    }
}
```