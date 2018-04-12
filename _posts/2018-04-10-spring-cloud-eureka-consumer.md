---
layout:     post
title:      Spring Cloud实战（二）服务消费者
subtitle:   构建微服务框架Spring Cloud之服务消费者Ribbon和Feign
date:       2018-04-10             
author:     jessehzx                
header-img: img/post-bg-os-metro.jpg
catalog: 	  true
tags:
    - Ribbon
    - Feign
        
---

> 版权声明：本文为 jessehzx 原创文章，支持转载，但必须在明确位置注明出处！！！

### 引子

前面一篇文章——[Spring Cloud实战（一）Eureka服务注册与发现](http://jessehzx.top/2018/04/08/spring-cloud-eureka/)，我们实现了一个服务注册中心(eureka server)和一个服务提供者(compute-service)，并且能将服务提供者注册到服务注册中心，那现在玩一玩服务消费者：Ribbon和Feign，我们都来尝试实现一下并看看它们带来的效果。

### 关于Ribbon

> 在Spring Cloud Netflix中，Ribbon是一个基于HTTP和TCP的客户端负载均衡器。

> 它通过客户端中配置的ribbonServerList服务端列表去轮询访问以达到负责均衡的效果。

> 当Ribbon与Eureka联合使用时，ribbonServerList会被DiscoveryEnabledNIWSServerList重写，扩展成从Eureka注册中心中获取服务端列表。同时它也会用NIWSDiscoveryPing来取代IPing，它将职责委托给Eureka来确定服务端是否已经启动。

> Ribbon工作时分为两步：第一步先选择Eureka Server, 它优先选择在同一个Zone且负载较少的Server；第二步再根据用户指定的策略，再从Server取到的服务注册列表中选择一个地址。其中Ribbon提供了多种策略，例如轮询、随机、根据响应时间加权等。

### 创建【服务消费者-Ribbon】

创建一个Spring boot工程，引入ribbon和eureka-client依赖，pom.xml如下：

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>top.jessehzx.cloud</groupId>
	<artifactId>service-consumer</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>eureka-consumer-ribbon</name>
	<description>Spring Cloud Eureka ribbon</description>

	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.0.1.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.8</java.version>
		<spring-cloud.version>Finchley.BUILD-SNAPSHOT</spring-cloud.version>
	</properties>

	<dependencies>
		<!-- 引入客户端负载均衡器ribbon -->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
		</dependency>

		<!-- 其实服务消费者仍然是客户端，要引入如下依赖，才能被服务注册中心发现 -->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
		</dependency>

		<!-- spring boot实现Java Web服务-->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
			<version>RELEASE</version>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>

	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>${spring-cloud.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>

</project>
```

在工程主类`EurekaConsumerRibbonApplication`中，定义REST客户端实例，代码如下：

```
package top.jessehzx.cloud.serviceconsumer;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

/**
 * @author jessehzx
 * @Date 2018/4/10
 */
@EnableDiscoveryClient
@SpringBootApplication
public class EurekaConsumerRibbonApplication {

    @Bean           // 定义REST客户端，RestTemplate实例
    @LoadBalanced   // 开启负载均衡
    RestTemplate restTemplate() {
        return new RestTemplate();
    }

	public static void main(String[] args) {
		SpringApplication.run(EurekaConsumerRibbonApplication.class, args);
	}
}
```

application.properties中定义了服务注册中心的地址、消费者服务的端口号、消费者服务的名称这些内容：

```
#应用名称
spring.application.name=ribbon-consumer

#端口号
server.port=9000

#注册中心的地址
eureka.client.serviceUrl.defaultZone=http://localhost:8761/eureka/
```

新建一个消费者服务的入口类：RibbonConsumerController，我们通过这个实例进行测试。消费者服务启动过程中，会从服务注册中心中拉最新的服务列表，当浏览器触发对应的请求，就会根据COMPUTE-SERVICE查找服务提供者的IP和端口号，然后发起调用。

```
package top.jessehzx.cloud.serviceconsumer;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

/**
 * @author jessehzx
 * @Date 2018/4/10
 */
@RestController
public class RibbonConsumerController {

    @Autowired
    private RestTemplate restTemplate;

    @RequestMapping("/add")
    public String add() {
        return restTemplate.getForEntity("http://COMPUTE-SERVICE/add?a=2&b=3", String.class).getBody();
    }

}
```

首先启动服务注册中心，然后分别启动两个服务提供者（第一个server.port=8888，第二个server.port=8889），然后启动服务消费者。

浏览器访问`localhost:8761`，发现都启动了，如下图：
![](https://ws2.sinaimg.cn/large/006tKfTcgy1fq7f5kb3shj31kw0qljyz.jpg)

> 如果你不知道怎么在Spring Boot功能启动多个实例，请[参阅本文](https://blog.csdn.net/forezp/article/details/76408139)

在浏览器里访问`localhost:9000/add`两次，可以看到请求有时候会在8888端口的服务，有时候会到8889的服务，那么使用Ribbon就看到负载均衡的效果了。如下两图所示：

![](https://ws3.sinaimg.cn/large/006tKfTcgy1fq7f06clb3j31kw05ktcw.jpg)
![](https://ws3.sinaimg.cn/large/006tKfTcgy1fq7f1e5sl6j31kw05oae9.jpg)

>**完整代码请点我查阅：[https://github.com/jessehzx/eureka-consumer-ribbon](https://github.com/jessehzx/eureka-consumer-ribbon)**


### 创建【服务消费者-Feign】

上一节中，使用类似restTemplate.getForEntity("http://COMPUTE-SERVICE/add?a=2&b=3", String.class).getBody()这样的语句进行服务间调用并非不可以，只是我们在服务化的过程中，希望跨服务调用能够看起来像本地调用，这也是我理解的Feign的使用场景。

创建一个spring boot工程，该工程的pom文件与上一节的类似，只是把ribbon的依赖换为feign的即可(spring-cloud-starter-openfeign)，代码如下：

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>top.jessehzx.cloud</groupId>
	<artifactId>service-consumer</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>eureka-consumer-feign</name>
	<description>Spring Cloud Eureka consumer feign</description>

	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.0.1.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.8</java.version>
		<spring-cloud.version>Finchley.BUILD-SNAPSHOT</spring-cloud.version>
	</properties>

	<dependencies>
		<!-- 引入feign -->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-openfeign</artifactId>
		</dependency>

		<!-- 其实服务消费者仍然是客户端，要引入如下依赖，才能被服务注册中心发现 -->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
		</dependency>

		<!-- spring boot实现Java Web服务-->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
			<version>RELEASE</version>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>

	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>${spring-cloud.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>

</project>
```

在程序启动类：EurekaConsumerFeignApplication上面加@EnableDiscoveryClient和@EnableFeignClients注解。

> 如果你的Feign接口定义跟你的启动类不在一个包名下，还需要制定扫描的包名@EnableFeignClients(basePackages = "top.jessehzx.cloud.xxx")

```
package top.jessehzx.cloud.serviceconsumer;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.openfeign.EnableFeignClients;

/**
 * @author jessehzx
 * @Date 2018/4/10
 */
@EnableFeignClients       // 用于启动Feign功能
@EnableDiscoveryClient    // 用于启动服务发现功能
@SpringBootApplication
public class EurekaConsumerFeignApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaConsumerFeignApplication.class, args);
	}
}
```

然后定义远程调用的接口。在top.jessehzx.cloud.serviceconsumer包下新建depend包，然后创建ComputeClient接口，使用@FeignClient("COMPUTE-SERVICE")注解修饰，COMPUTE-SERVICE就是服务提供者的名称，然后定义要使用的服务，代码如下：

```
package top.jessehzx.cloud.serviceconsumer.depend;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RequestParam;

/**
 * @author jessehzx
 * @Date 2018/4/10
 */
@FeignClient("COMPUTE-SERVICE")
public interface ComputeClient {

    @RequestMapping(method = RequestMethod.GET, value = "/add")
    Integer add(@RequestParam(value = "a") Integer a, @RequestParam(value = "b") Integer b);

}
```

新建测试类FeignConsumerController。由于上面的接口使用了@FeignClient，它会通知Feign组件对该接口进行代理(不需要编写接口实现)，所以使用者可直接通过@Autowired注入使用。代码如下：

```
package top.jessehzx.cloud.serviceconsumer;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;
import top.jessehzx.cloud.serviceconsumer.depend.ComputeClient;

/**
 * @author jessehzx
 * @Date 2018/4/10
 */
@RestController
public class FeignConsumerController {
	
	// 注入服务提供者,远程的Http服务
    @Autowired
    private ComputeClient computeClient;

    @RequestMapping(value = "/add", method = RequestMethod.GET)
    public Integer add() {
        return computeClient.add(2, 3);
    }

}
```

application.properties不变，改下name的名称即可，如下：

```
#应用名称
spring.application.name=feign-consumer

#端口号
server.port=9000

#注册中心的地址
eureka.client.serviceUrl.defaultZone=http://localhost:8761/eureka/
```

启动步骤和Ribbon的一样：

- 先启动服务注册中心eureka-register
- 再启动两个服务提供者eureka-provider(一个8888端口，一个8889端口)
- 启动服务消费者eureka-consumer-feign

访问`localhost:8761`，得到如下图所示，说明正常work:
![](https://ws2.sinaimg.cn/large/006tKfTcgy1fq7mdb5hccj31kw0qdn4s.jpg)

访问localhost:9000/add几次，也可以看到两个服务提供者都已经收到了消费者发来的请求。

![](https://ws2.sinaimg.cn/large/006tKfTcgy1fq7mk1c4ndj31kw05sn1p.jpg)

![](https://ws2.sinaimg.cn/large/006tKfTcgy1fq7mkpa5csj31kw064wj5.jpg)

总结：我们通过Feign以接口和注解配置的方式，轻松实现了对compute-service服务的绑定，这样我们就可以在本地应用中像本地服务一下的调用它，并且做到了客户端均衡负载。

>**完整代码请点我查阅：[https://github.com/jessehzx/eureka-consumer-feign](https://github.com/jessehzx/eureka-consumer-feign)**
