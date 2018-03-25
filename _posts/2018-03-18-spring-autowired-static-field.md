---
layout:     post
title:      Spring中使用@Autowired注解静态变量 
subtitle:   分析为什么Spring中不能自动装配静态变量，并提出4种解决方案。
date:       2018-03-18             
author:     jessehzx                
header-img: 
catalog: true
tags:
    - Spring
    - 后台开发    
---

>版权声明：本文为 jessehzx 原创文章，支持转载，但必须在明确位置注明出处！！！

### 问题

最近项目小组在重新规划工程的业务缓存，其中涉及到部分代码重构，过程中发现有些工具类中的静态方法需要依赖别的对象实例（该实例已配置在xml成bean，非静态可以用@Autowired加载正常使用），而我们知道，类加载后静态成员是在内存的共享区，静态方法里面的变量必然要使用静态成员变量，这就有了如下代码：

```
@Component
public class TestClass {

    @Autowired
    private static AutowiredTypeComponent component;

    // 调用静态组件的方法
    public static void testMethod() {
        component.callTestMethod()；
    }
    
}
```
编译正常，但运行时报java.lang.NullPointerException: null异常，显然在调用testMethod()方法时，component变量还没被初始化，报NPE。

### 原因

在`Springframework`里，我们不能@Autowired一个静态变量，使之成为一个Spring bean。为什么？其实很简单，因为当类加载器加载静态变量时，Spring上下文尚未加载。所以类加载器不会在bean中正确注入静态类，并且会失败。
    

### 解决方案

- 方式一

将@Autowired 注解到类的构造函数上。很好理解，Spring扫描到AutowiredTypeComponent的bean，然后赋给静态变量component。示例如下：

```
@Component
public class TestClass {

    private static AutowiredTypeComponent component;

    @Autowired
    public TestClass(AutowiredTypeComponent component) {
        TestClass.component = component;
    }

    // 调用静态组件的方法
    public static void testMethod() {
        component.callTestMethod()；
    }
    
}
```
- 方式二

给静态组件加setter方法，并在这个方法上加上@Autowired。Spring能扫描到AutowiredTypeComponent的bean，然后通过setter方法注入。示例如下：

```
@Component
public class TestClass {

    private static AutowiredTypeComponent component;

    @Autowired
    public void setComponent(AutowiredTypeComponent component){
        TestClass.component = component;
    }
    
    // 调用静态组件的方法
    public static void testMethod() {
        component.callTestMethod()；
    }
    
}
```

- 方式三

定义一个静态组件，定义一个非静态组件并加上@Autowired注解，再定义一个初始化组件的方法并加上@PostConstruct注解。这个注解是JavaEE引入的，作用于servlet生命周期的注解，你只需要知道，用它注解的方法在构造函数之后就会被调用。示例如下：

```
@Component
public class TestClass {

   private static AutowiredTypeComponent component;

   @Autowired
   private AutowiredTypeComponent autowiredComponent;

   @PostConstruct
   private void beforeInit() {
      component = this.autowiredComponent;
   }
   
   // 调用静态组件的方法
   public static void testMethod() {
      component.callTestMethod();
   }
   
}
```

- 方式四
 
直接用Spring框架工具类获取bean，定义成局部变量使用。但有弊端：如果该类中有多个静态方法多次用到这个组件则每次都要这样获取，个人不推荐这种方式。示例如下：

```
public class TestClass {
    
    // 调用静态组件的方法
   public static void testMethod() {
      AutowiredTypeComponent component = SpringApplicationContextUtil.getBean("component");
      component.callTestMethod();
   }
    
}

```

### 总结

- 在上面的代码示例中，我每个类都加了@Component注解，其实可以根据需要进行变更，比如这个类是处理业务逻辑，可以换成@Service；这个类是处理请求进行转发或重定向的，可以换成@Controller（是Spring-mvc的注解）；这个类是专门用来操作Dao的就@Repository。Spring的注解帮你做了一件很有意义的事：就是它们对应用进行了分层，这样就能将请求处理、业务逻辑处理、数据库操作处理分离出来，为代码解耦，也方便了项目的开发和维护。

- Spring容器bean加载机制用到了Java的反射，这里先不作赘述，以后会专门写一篇文章来总结Java反射在Spring的IoC和AoP中的应用。
