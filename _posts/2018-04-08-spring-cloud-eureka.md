---
layout:     post
title:      Spring Cloud实战（一）Eureka服务注册与发现
subtitle:   构建微服务框架Spring Cloud之注册中心Eureka
date:       2018-04-08             
author:     jessehzx                
header-img: img/post-bg-os-metro.jpg
catalog: 	  true
tags:
    - Spring
        
---

>版权声明：本文为 jessehzx 原创文章，支持转载，但必须在明确位置注明出处！！！

### 背景知识
如果你有如下知识储备，阅读本文可能会更顺利一些。不懂的话也没关系，先跟着我一步一步操作，遇到困惑的地方可以Google或留言。

- 微服务架构
- Spring Boot
- Maven

### Spring Cloud简介
Spring Cloud是一个为开发者提供实现分布式系统特性的工具。例如，配置管理、服务发现、熔断器、智能路由、微代理（micro-proxy）、控制总线、one-time tokens、全局锁（global locks）、选举算法（leadership election）、分布式会话和集群状态管理（cluster status）等。

Spring Cloud下有很多子工程：

- Spring Cloud Config：依靠git仓库实现的中心化配置管理。配置资源可以映射到Spring的不同开发环境中，但是也可以使用在非Spring应用中。
- Spring Cloud Netflix：不同的Netflix OSS组件的集合：Eureka、Hystrix、Zuul、Archaius等。
- Spring Cloud Bus：事件总线，利用分布式消息将多个服务连接起来。非常适合在集群中传播状态的改变事件（例如：配置变更事件）
- Spring Cloud Consul：服务发现和配置管理，由Hashicorp团队开发。

我决定先从Spring Cloud Netflix看起，它基本上是对Netflix公司的一系列开源产品进行包装，它为Spring Boot应用提供了自配置的Netflix OSS整合。通过一些简单的注解，开发者就可以在应用中快速地配置常用模块并构建庞大的分布式系统。它主要提供的模块包括：服务发现（Eureka），断路器（Hystrix），智能路有（Zuul），客户端负载均衡（Ribbon）等。

本文的核心内容就是服务发现模块：Eureka。下面我们动手来做一些尝试。
需要创建三个角色：服务注册中心、服务提供者、服务消费者。


### 创建【服务注册中心】
在IntelliJ IDEA中创建一个Spring Cloud工程，引入Eureka-Server包依赖，pom.xml文件如下：

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.example.springcloud</groupId>
    <artifactId>service-register</artifactId>
    <version>1.0-SNAPSHOT</version>


    <!-- spring boot的parent 配置文件，有大部分spring boot需要用的Jar包 -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.1.RELEASE</version>
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <java.version>1.8</java.version>
    </properties>

    <!-- spring boot的maven打包插件 -->
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

    <dependencies>
        <!-- eureka-server -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka-server</artifactId>
        </dependency>

        <!-- spring boot test-->
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
                <version>Camden.SR5</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

通过@EnableEurekaServer注解启动一个服务注册中心提供给其他应用进行对话。这一步非常的简单，只需要在一个普通的Spring Boot应用中添加这个注解就能开启此功能，代码如下：

```
package top.jessehzx;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

/**
 * Created by IntelliJ IDEA.
 * User: jessehzx
 * Date: 2018/04/08
 * Time: 22:50
 */

@EnableEurekaServer //通过@EnableEurekaServer启动一个服务注册中心给其他应用使用
@SpringBootApplication
public class EurekaServiceRegisterApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaServiceRegisterApplication.class, args);
    }
}
```

在application.properties中还需要增加如下配置，才能创建一个真正可以使用的服务注册中心。

```
#注册服务的端口号
server.port=8761

#是否需要注册到注册中心，因为该项目本身作为服务注册中心，所以为false
eureka.client.register-with-eureka=false
#是否需要从注册中心获取服务列表，原因同上，为false
eureka.client.fetch-registry=false
#注册服务器的地址：服务提供者和服务消费者都要依赖这个地址
eureka.client.service-url.defaultZone=http://localhost:${server.port}/eureka

logging.level.com.netflix.eureka=OFF
logging.level.com.netflix.discovery=OFF
```
直接点击main()主函数右键Run，快捷键`⌃+⇧+R`，启动注册服务，并浏览器访问：`http://localhost:8761`，就可以看到如下界面。但其中还没有发现任何服务。

![](https://ws2.sinaimg.cn/large/006tNc79gy1fq5n5hxs4hj31kw0vwn4p.jpg)

### 创建【服务提供者】
创建一个Spring Boot工程，代表服务提供者，该服务提供者会暴露一个加法服务，接受客户端传来的加数和被加数，并返回两者的和。pom.xml文件如下：

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.example.springcloud</groupId>
    <artifactId>service-client</artifactId>
    <version>1.0-SNAPSHOT</version>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.1.RELEASE</version>
        <relativePath/> 
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <java.version>1.8</java.version>
    </properties>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka-server</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Camden.SR5</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

其中的关键在于spring-cloud-starter-eureka这个Jar包，其中包含了eureka的客户端实现。

在src/main/java/hello下创建工程的主类EurekaServerProducerApplication，使用@EnableDiscoveryClient注解修饰，该注解在服务启动的时候，可以触发服务注册的过程，向配置文件中指定的服务注册中心（Eureka-Server）的地址注册自己提供的服务。EurekaServerProducerApplication的源码如下：

```
package top.jessehzx;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

/**
 * Created by IntelliJ IDEA.
 * User: jessehzx
 * Date: 2018/04/08
 * Time: 22:55
 */
@EnableDiscoveryClient
@SpringBootApplication
public class EurekaServerProducerApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaServerProducerApplication.class, args);
    }

}
```

application.xml配置如下：

```
#服务提供者的名字
spring.application.name=compute-service

#服务提供者的端口号
server.port=8888

#服务注册中心的地址
eureka.client.serviceUrl.defaultZone=http://localhost:8761/eureka/
```

服务提供者的基本框架搭好后，需要实现服务的具体内容，在ServiceInstanceRestController类中实现，它的具体代码如下：

```
package hello;


import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.discovery.DiscoveryClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

/**
 * Created by IntelliJ IDEA.
 * User: duqi
 * Date: 2017/3/2
 * Time: 20:36
 */
@RestController
public class ServiceInstanceRestController {

    private static final Logger logger = LoggerFactory.getLogger(ServiceInstanceRestController.class);

    @Autowired
    private DiscoveryClient discoveryClient; //服务发现客户端

    @GetMapping(value = "/add")
    public Integer add(@RequestParam Integer a, @RequestParam Integer b) {
        ServiceInstance instance = discoveryClient.getLocalServiceInstance();
        Integer r = a + b;
        logger.info("/add, host:" + instance.getHost() + ", service_id:" + instance.getServiceId() + ", result:" + r);
        return r;
    }
}
```

先启动服务注册中心，再启动服务提供者，当你再次通过浏览器访问：`http://localhost:8761`时，是不是发现服务提供者已经注册到服务注册中心了。

![](https://ws3.sinaimg.cn/large/006tNc79gy1fq5nii4ssaj31kw0vz10q.jpg)

