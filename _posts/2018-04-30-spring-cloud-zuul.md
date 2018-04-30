---
layout:     post
title:      Spring Cloud实战（五）服务网关Zuul
subtitle:   基于微服务框架Spring Cloud服务网关Zuul实战演练
date:       2018-04-30             
author:     jessehzx                
header-img: img/post-bg-os-metro.jpg
catalog: 	  true
tags:
    - Spring Cloud
    - Zuul
        
---

> 版权声明：本文为 jessehzx 原创文章，支持转载，但必须在明确位置注明出处！！！

### 简介
Spring Cloud Zuul路由是微服务架构的不可或缺的一部分，它提供动态路由，监控，弹性，安全等的边缘服务。Zuul是Netflix出品的一个基于JVM路由和服务端的负载均衡器。

下面我们通过代码来了解Zuul是如何工作的。

简单使用

**1、添加依赖**

```
<dependencies>
	<dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-starter-netflix-zuul</artifactId>
	</dependency>
</dependencies>
```
引入spring-cloud-starter-netflix-zuul包。

**2、配置文件**

```
spring.application.name=gateway-service-zuul
server.port=8888

#这里的配置表示，访问/it/** 直接重定向到http://jessehzx.top/**
zuul.routes.gateway.path=/it/**
zuul.routes.gateway.url=http://jessehzx.top/
```

**3、启动类**

```
package top.jessehzx.cloud.gatewayservicezuul;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.zuul.EnableZuulProxy;

@EnableZuulProxy
@SpringBootApplication
public class GatewayServiceZuulApplication {

	public static void main(String[] args) {
		SpringApplication.run(GatewayServiceZuulApplication.class, args);
	}
}

```

启动类添加@EnableZuulProxy，支持网关路由。

**4、测试**

启动gateway-service-zuul项目，在浏览器中访问：http://localhost:8888/it/tags，看到页面返回了：http://jessehzx.top/tags/ 页面的信息，如下：

![](https://ws1.sinaimg.cn/large/006tKfTcgy1fquoh2w6fqj31kw0khnoo.jpg)

说明访问gateway-service-zuul的请求自动转发到了http://jessehzx.top/，并且将结果返回。

### 服务化

通过url映射的方式来实现Zuul的转发有局限性，比如每增加一个服务就需要配置一条内容，另外后端的服务如果是动态来提供，就不能采用这种方案来配置了。实际上在实现微服务架构时，服务名与服务实例地址的关系在eureka server中已经存在了，所以只需要将Zuul注册到eureka server上去发现其他服务，就可以实现对serviceId的映射。

我们结合示例来说明，在上面示例项目gateway-service-zuul的基础上来改造。

**1、添加依赖**

```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

增加spring-cloud-starter-netflix-eureka-client包，添加对eureka的支持。

**2、配置文件**

配置修改为：

```
spring.application.name=gateway-service-zuul
server.port=8888

zuul.routes.api-a.path=/provider/**
zuul.routes.api-a.serviceId=compute-service

eureka.client.serviceUrl.defaultZone=http://localhost:8761/eureka/
```

**3、测试**

依次启动 eureka-register、computer-service、gateway-service-zuul，先访问：http://localhost:8761，看到了我们第一篇文章刚学服务注册中心时熟悉的界面，如下图：

![](https://ws4.sinaimg.cn/large/006tKfTcgy1fqv1jlavvaj31kw075go3.jpg)

再访问：http://localhost:8888/provider/add?a=2&b=3，返回：5。

![](https://ws4.sinaimg.cn/large/006tKfTcgy1fqv1ljr1g4j30u004agly.jpg)

说明访问gateway-service-zuul的请求自动转发到了compute-service，并且将结果返回。

### 实现负载均衡

为了更好的模拟服务集群，我们修改compute-service项目端口为9000，controller代码修改如下：

```
@RequestMapping("/add")
public Integer add(@RequestParam Integer a, @RequestParam Integer b) {
    ServiceInstance instance = serviceInstance();
    Integer res = (a + b) * 2; // 这里是为了演示负载均衡的效果，实际不会这么写
    LOGGER.info("/add, host:" + instance.getHost() + ", service_id:" + instance.getServiceId() + ", result:" + res);
    return res;
}
```
修改完成后启动它（相当于多启动了一个服务提供者），重启gateway-service-zuul。测试多次访问http://localhost:8888/provider/add?a=2&b=3，依次返回：

```
5
10
5
10
...
```

说明通过zuul成功调用了compute-service服务并且做了均衡负载。

### 网关的默认路由规则

但是如果后端服务多达十几个的时候，每一个都这样配置也挺麻烦的，Spring Cloud Zuul已经帮我们做了默认配置。默认情况下，Zuul会代理所有注册到Eureka Server的微服务，并且Zuul的路由规则如下：http://ZUUL_HOST:ZUUL_PORT/微服务在Eureka上的serviceId/**会被转发到serviceId对应的微服务。

我们注释掉gateway-service-zuul项目中关于路由的配置：

```
#zuul.routes.api-a.path=/provider/**
#zuul.routes.api-a.serviceId=compute-service
```

重新启动后，访问http://localhost:8888/compute-service/add?a=2&b=3，测试返回结果和上述示例相同，说明Spirng Cloud Zuul默认已经提供了转发功能。

至此，Spring Cloud Zuul的基本使用我们就介绍完了。

>**完整代码请点我查阅：[https://github.com/jessehzx/gateway-service-zuul](https://github.com/jessehzx/gateway-service-zuul)**






