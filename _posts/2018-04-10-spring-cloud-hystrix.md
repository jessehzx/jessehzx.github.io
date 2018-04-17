---
layout:     post
title:      Spring Cloud实战（三）断路器
subtitle:   基于微服务框架Spring Cloud服务消费者Ribbon和Feign构建断路器Hystrix
date:       2018-04-17             
author:     jessehzx                
header-img: img/post-bg-os-metro.jpg
catalog: 	  true
tags:
    - Hystrix
        
---

> 版权声明：本文为 jessehzx 原创文章，支持转载，但必须在明确位置注明出处！！！

### 引子

- [Spring Cloud实战（一）Eureka服务注册与发现](http://jessehzx.top/2018/04/08/spring-cloud-eureka/)
- [Spring Cloud实战（二）服务消费者](http://jessehzx.top/2018/04/10/spring-cloud-eureka-consumer/)

通过前面两篇文章的介绍，我们已经能够使用Spring Cloud开发出一个简单的系统了. 这篇文章里，我们将关注点转移到如何提高微服务系统的容错性(Fault Tolerance)，并了解如何借助Hystrix开发健壮的微服务。

以往我们在开发单体应用，或者调用RPC服务的时候，可能没有考虑太多目标服务调用失败的情况，经常一个Try/Catch加上打印日志就解决了。但是在微服务系统中，这种处理方法会给系统的稳定性带来很大隐患。举个例子，假设我们系统中下单的功能要依赖50个服务，每个服务正常响应的概率为99.99%，如果我们不做容错处理，只要任意一个服务没有响应下单就失败的话，我们下单成功的概率为99.99的50次方=99.50%，即日订单量为1W的话，50个会出现下单失败，这还是建立在依赖服务稳定性很高的情况下(4个9)。但是服务调用失败引起的问题不仅仅是这么简单，在分布式环境下，一个服务的调用失败可能会使其他被依赖服务发生延迟和超时，而且这个影响会很快扩散到其他服务，从而引发整个系统的雪崩(Avalanche)。

所以，引入了断路器（Hystrix）模式。

### 在Ribbon中使用Hystrix

在上一节[Spring Cloud实战（二）服务消费者](http://jessehzx.top/2018/04/10/spring-cloud-eureka-consumer/)的eureka-consumer-ribbon工程中，引入hystrix依赖，如下：

```
<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

在主类中加上`@EnableCircuitBreaker`注解，代码如下：

```
package top.jessehzx.cloud.serviceribbonhystrix;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

@EnableCircuitBreaker
@EnableDiscoveryClient
@SpringBootApplication
public class ServiceRibbonHystrixApplication {

	@Bean           // 定义REST客户端，RestTemplate实例
	@LoadBalanced   // 开启负载均衡
	RestTemplate restTemplate() {
		return new RestTemplate();
	}

	public static void main(String[] args) {
		SpringApplication.run(ServiceRibbonHystrixApplication.class，args);
	}
}

```

我们新建一个类`ComputeService`，把工程主类`EurekaConsumerRibbonApplication`中业务代码迁移过来，并在add()方法上加上注解`@HystrixCommand(fallbackMethod = "addFallback")`，定义addFallback()方法，用来对fallback的情况执行返回，代码如下：

```
package top.jessehzx.cloud.serviceribbonhystrix;

import com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

/**
 * @author jessehzx
 * @date 2018/4/16
 */
@Service
public class ComputeService {

    @Autowired
    private RestTemplate restTemplate;

    //    @RequestMapping("/add")
    @HystrixCommand(fallbackMethod = "addFallback")
    public String add() {
        return restTemplate.getForEntity("http://COMPUTE-SERVICE/add?a=2&b=3"，String.class).getBody();
    }

    public String addFallback() {
        return "sorry，error!";
    }

}
```

在消费者服务的入口类：RibbonConsumerController中，我们通过`Autowired`注解把ComputeService注入进来，代码如下：

```
package top.jessehzx.cloud.serviceribbonhystrix;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * @author jessehzx
 * @Date 2018/4/10
 */
@RestController
public class RibbonConsumerController {

    @Autowired
    private ComputeService computeService;

    @RequestMapping("/add")
    public String add() {
        return computeService.add();
    }

}
```

好了，现在我们启动他们来作验证：

- 启动服务注册中心eureka-register
- 启动服务提供者compute-service
- 启动该断路器工程service-ribbon-hystrix

浏览器访问`http://localhost:8761/`，可以看到启动了以上服务：

![](https://ws3.sinaimg.cn/large/006tKfTcgy1fqfes834znj31kw0qcah0.jpg)

访问`http://localhost:9000/add`，可以看到服务正常使用：

![](https://ws4.sinaimg.cn/large/006tKfTcgy1fqfevycxxlj30rg03wdg0.jpg)

停掉服务提供者compute-service，再次访问`http://localhost:9000/add`，得到如下图：

![](https://ws4.sinaimg.cn/large/006tKfTcgy1fqffnzfdwuj30ua0460t0.jpg)

>**完整代码请点我查阅：[https://github.com/jessehzx/service-ribbon-hystrix](https://github.com/jessehzx/service-ribbon-hystrix)**


### Feign中引入Hystrix

这里不需要像上面那样在pox.xml中加入Hystrix的依赖，因为如果你阅读了Feign源码，就会发现Feign中已经集成了断路器Hystrix。所以我们直接通过注解的方式将它使用起来即可。

我们在Eureka-consumer-feign工程进行改造，在接口ComputeClient的注解`@FeignClient`中加上`fallback=ComputeClientFallback.class`，代码如下：

```
package top.jessehzx.cloud.servicefeignhystrix.depend;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RequestParam;
import top.jessehzx.cloud.servicefeignhystrix.ComputeClientFallback;

/**
 * @author jessehzx
 * @date 2018/4/16
 */
@FeignClient(value = "COMPUTE-SERVICE"，fallback = ComputeClientFallback.class)
public interface ComputeClient {

    @RequestMapping(method = RequestMethod.GET，value = "/add")
    Integer add(@RequestParam(value = "a") Integer a，@RequestParam(value = "b") Integer b);

}

```

再新建一个类`ComputeClientFallback`，该类实现ComputeClient接口。代码如下：

```
package top.jessehzx.cloud.servicefeignhystrix;

import org.springframework.stereotype.Component;
import org.springframework.web.bind.annotation.RequestParam;
import top.jessehzx.cloud.servicefeignhystrix.depend.ComputeClient;

/**
 * @author jessehzx
 * @date 2018/4/16
 */
@Component
public class ComputeClientFallback implements ComputeClient{

    @Override
    public Integer add(@RequestParam(value = "a") Integer a，@RequestParam(value = "b") Integer b) {
        return -1;
    }
}

```

好了，现在我们启动他们来作验证：

- 启动服务注册中心eureka-register
- 启动服务提供者compute-service
- 启动该断路器工程service-feign-hystrix

浏览器访问`http://localhost:8761/`，可以看到启动了以上服务：

![](https://ws3.sinaimg.cn/large/006tKfTcgy1fqfes834znj31kw0qcah0.jpg)

访问`http://localhost:9000/add`，可以看到服务正常使用：

![](https://ws4.sinaimg.cn/large/006tKfTcgy1fqfevycxxlj30rg03wdg0.jpg)

停掉服务提供者compute-service，再次访问`http://localhost:9000/add`，得到如下图：

![](https://ws3.sinaimg.cn/large/006tKfTcgy1fqfeodyxxbj30ss04uweo.jpg)

可以看到，即使我的服务提供者由于某种原因出故障了，不能正常提供服务了，但是我加入了断路器Hystrix，就能通过它把请求打到异常处理类并返回相应结果，而不用try{}catch{}了。

>**完整代码请点我查阅：[https://github.com/jessehzx/service-ribbon-hystrix](https://github.com/jessehzx/service-ribbon-hystrix)**
