---
layout:     post
title:      Spring Cloud实战（1）- Eureka注册中心
subtitle:   微服务框架Spring Cloud之注册中心Eureka
date:       2018-04-07             
author:     jessehzx                
header-img: img/post-bg-os-metro.jpg
catalog: 	  true
tags:
    - Spring
        
---

>版权声明：本文为 jessehzx 原创文章，支持转载，但必须在明确位置注明出处！！！

### 背景知识
如果你有如下几点知识储备，阅读本文可能会更顺利一些。

- Spring Boot知识
- Maven知识
- Docker知识

### 简介
Spring Cloud为开发者提供了实现分布式系统特性的工具。例如，配置管理、服务发现、熔断器、智能路由、微代理（micro-proxy）、控制总线、one-time tokens、global locks、选举算法（leadership election）、分布式会话和cluster status。通过Spring Cloud开发者可以快速建立具有分布式特性的应用。

### 项目简介

- 项目地址：[点我](https://github.com/jessehzx)
- 开发工具：IntelliJ IDEA/Maven
- 实施部署工具：Docker

### 创建根项目
我们通过Maven创建一个POM项目，项目名称叫做spring-cloud-poc。pom.xml文件如下：

```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>top.jessehzx</groupId>
    <artifactId>cloud</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>pom</packaging>
    <name>cloud</name>
    <url>http://maven.apache.org</url>
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <spring-cloud.version>Dalston.SR3</spring-cloud.version>
        <spring-boot.version>1.5.7.RELEASE</spring-boot.version>
    </properties>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>junit</groupId>
                <artifactId>junit</artifactId>
                <version>4.12</version>
                <scope>test</scope>
            </dependency>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <type>pom</type>
                <version>${spring-boot.version}</version>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                    <version>${spring-boot.version}</version>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
</project>
```

我们在根项目中配置了子项目需要的依赖包和插件。下面我们开始创建一个Eureka Server项目。

### 创建Eureka 注册中心项目
我们在spring-cloud-poc项目下，创建一个eureka子项目。该子项目pom.xml文件中，我们引入Eureka Server所必须的依赖。

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <artifactId>eureka</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>
    <name>eureka</name>
    <description>Eureka Server</description>
    <parent>
        <groupId>top.jessehzx</groupId>
        <artifactId>cloud</artifactId>
        <version>1.0-SNAPSHOT</version>
        <!--<relativePath/> <!– lookup parent from repository –>-->
    </parent>
    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka-server</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <fork>true</fork>
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

### 创建注册中心启动类
我们采用Spring Boot风格创建一个启动类top.jessehzx.eureka.EurekaApplication，代码如下：

```
package top.jessehzx.eureka;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;
@SpringBootApplication
@EnableEurekaServer
public class EurekaApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaApplication.class, args);
    }
}
```

### 创建一个默认配置文件
我们给程序创建一个默认的配置文件application.yml。

```
server:
  port: 8761
eureka:
  server:
    enableSelfPreservation: true
#    eviction-interval-timer-in-ms: 5000
  instance:
#    perferIpAddress: true
#    instance-id: ${spring.cloud.client.ipAddress}:${server.port}
    hostname: localhost
    contextroot: eureka
  client:
    registerWithEureka: true
    fetchRegistry: false
    healthcheck:
      enabled: true
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/${eureka.instance.contextroot}/
spring:
  application:
    name: Eureka Server
```

**注意： 
我们这里创建的是程序的默认配置。实际运行过程中，会根据运行环境指定不同的配置文件来覆盖默认配置文件的数据。参考Spring Boot开发框架配置文件相关的知识。**

配置文件的说明如下：

- 服务名称

我们指定如下的参数，即为Eureka的管理台中的service name，服务名。

```
spring:
  application:
    name: Eureka Server
```

- Eureka 配置

```
eureka:
  server:
    enableSelfPreservation: true #为true的话，在所有注册服务的元数据中，包含注册中心的注册数据。
#    eviction-interval-timer-in-ms: 5000
  instance:
#    perferIpAddress: true
#    instance-id: ${spring.cloud.client.ipAddress}:${server.port}
    hostname: localhost # 运行程序的服务器的hostname
    contextroot: eureka 
  client:
    registerWithEureka: true # 指定客户端使用Eureka注册中心的形式进行注册
    fetchRegistry: false 
    healthcheck:
      enabled: true # 开启服务心跳检查
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/${eureka.instance.contextroot}/ 
```
一个服务注册中心我们也可以看做是某一个服务的实例，所以服务注册中心实例也可以注册到另外一个服务注册中心实例上。通过如下配置实现：

```
eureka:
  client:
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/${eureka.instance.contextroot}/ 
```

多个服务注册中心互相注册，就形成一个replication模式的高可用注册中心集群。两个互相注册的注册中心实例，关于注册服务的元数据是互为副本的。当一个注册中心实例为down的状态时，不会影响到其他服务的注册和发现。

### Docker容器部署
我们回过头来看一下，我们创建了一个启动类EurekaApplication和一个默认配置文件application.yml，然后就完成了一个Eureka Server注册中心的的程序。是不是很惊喜？就是这么简单！

我们完成程序后，需要部署测试一下我们的注册中心。我们采用Docker虚拟容器来测试我们的程序。

我们在spring-cloud-poc/eureka/config目录下创建两个配置文件，分别用来作为两个Eureka Server容器的实际配置文件。

```
# application-prd.1.yml
eureka:
  instance:
    hostname: eureka-1
  client:
    serviceUrl:
        defaultZone: http://eureka-2:8761/eureka/
```   
        
```        
# application-prd.2.yml
eureka:
  instance:
    hostname: eureka-2
  client:
      serviceUrl:
          defaultZone: http://eureka-1:8761/eureka/
```
          
我们可以看出我们即将部署的两个Eureka Server容器eureka-1和eureka-2它们互相注册，成为彼此的副本。

我们在spring-cloud-poc目录下创建一个docker-compose.yml文件。

```
version: '3'
services:
  eureka-1:
    image: openjdk:8-jdk-alpine
    entrypoint:
      - "sh"
      - "-c"
      - "java $JAVA_OPTS -Djava.security.egd=file:/dev/./urandom -jar app.jar --spring.profiles.active=prd"
    container_name: eureka-1
    volumes:
      - "/tmp"
      - "./eureka/target/eureka-0.0.1-SNAPSHOT.jar:/app.jar"
      - "/config"
      - "./eureka/config/application-prd.1.yml:/config/application-prd.yml" #实际配置文件挂载
#      - "./config/log4j2.xml:/config/log4j2.xml"
    environment:
      JAVA_OPTS: -XX:MetaspaceSize=64m -XX:MaxMetaspaceSize=512m
    ports:
          - "8761:8761"
    networks:
          mynet:
            ipv4_address: 172.19.0.2
    extra_hosts:
      - "eureka-1:172.19.0.2"
      - "eureka-2:172.19.0.3"
  eureka-2:
      image: openjdk:8-jdk-alpine
      entrypoint:
        - "sh"
        - "-c"
        - "java $JAVA_OPTS -Djava.security.egd=file:/dev/./urandom -jar app.jar --spring.profiles.active=prd"
      container_name: eureka-2
      volumes:
        - "/tmp"
        - "./eureka/target/eureka-0.0.1-SNAPSHOT.jar:/app.jar"
        - "/config"
        - "./eureka/config/application-prd.2.yml:/config/application-prd.yml" #实际配置文件挂载
  #      - "./config/log4j2.xml:/config/log4j2.xml"
      environment:
        JAVA_OPTS: -XX:MetaspaceSize=64m -XX:MaxMetaspaceSize=512m
      ports:
            - "8762:8761"
      networks:
            mynet:
              ipv4_address: 172.19.0.3
      extra_hosts:
        - "eureka-1:172.19.0.2"
        - "eureka-2:172.19.0.3"
networks:
  mynet:
    driver: bridge
#    enable_ipv6: false
    ipam:
      driver: default
      config:
        - subnet: 172.19.0.0/16
#        - gateway: 172.18.0.1
```

了解Docker的读者可能注意到了，在部署过程中我们虚拟了一个桥接网络，并为容器分别指定了IP地址。这样做的目的是为了在Eureka Server的管理页面中我们能够直观的查看注册信息数据。我们将eureka-1和eureka-2的8761端口分别暴露到宿主机的8761和8762端口。我们将通过浏览器来访问这两个端口上的控制台页面。

### 验证
使用mvn clean install命令编译整个spring-cloud-poc项目。然后在spring-cloud-poc目录下运行docker-compose up -d命令，启动docker-compose.yml中配置的容器。

通过浏览器访问：

```
http://localhost:8761/eureka
http://localhost:8762/eureka
```