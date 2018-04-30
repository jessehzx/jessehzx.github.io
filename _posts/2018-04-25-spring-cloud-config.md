---
layout:     post
title:      Spring Cloud实战（四）配置中心
subtitle:   基于微服务框架Spring Cloud分布式配置中心config的git示例
date:       2018-04-25             
author:     jessehzx                
header-img: img/post-bg-os-metro.jpg
catalog: 	  true
tags:
    - Config
        
---

> 版权声明：本文为 jessehzx 原创文章，支持转载，但必须在明确位置注明出处！！！

### 简介
在分布式系统中，由于服务数量巨多，为了方便服务配置文件统一管理，实时更新，所以需要分布式配置中心组件。

在我们了解Spring Cloud Config之前，我们可以先想想一个配置中心应该提供哪些核心功能？

- 提供服务端和客户端支持
- 集中管理各环境的配置文件
- 配置文件修改之后，可以快速的生效
- 可以进行版本管理
- 支持大的并发查询
- 支持各种语言

Spring Cloud Config可以完美的支持以上所有的需求。

在spring cloud config 组件中，分两个角色，一是config server，二是config client。server提供配置文件的存储、以接口的形式将配置文件的内容提供出去，client通过接口获取数据、并依据此数据初始化自己的应用。它支持配置服务放在配置服务的内存中（即本地），也支持放在远程git或svn仓库中，默认情况下使用git，接下来我们以git为例进行实战演示。

首先在github上创建了一个工程，我取名为SpringcloudConfig，在工程下面创建一个文件夹config用来存放配置文件，为了模拟生产环境，我们创建以下三个配置文件：conf_test.properties、conf_dev.properties、conf_pro.properties，里面文件内容分别对应jessehzx.habit=coding test、jessehzx.habit=coding dev、jessehzx.habit=coding pro，下面我们构建config server。

### 构建config server

**1、新建工程，引入依赖**

创建一个spring boot项目，取名为config-server，其pom.xml:

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>top.jessehzx.cloud</groupId>
	<artifactId>config-service</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>config-service</name>
	<description>Spring Cloud Config Server</description>

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
		<spring-cloud.version>Finchley.RC1</spring-cloud.version>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-config-server</artifactId>
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

**2、启动类**

在程序的入口Application类加上@EnableConfigServer注解开启配置服务器的功能，代码如下：

```
package top.jessehzx.cloud.configservice;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;

@EnableConfigServer
@SpringBootApplication
public class ConfigServiceApplication {

	public static void main(String[] args) {
		SpringApplication.run(ConfigServiceApplication.class, args);
	}
}

```

**3、配置文件**

需要在程序的配置文件application.properties文件配置以下：

```
spring.application.name=config-server
server.port=8888

spring.cloud.config.server.git.uri=https://github.com/jessehzx/SpringcloudConfig/
spring.cloud.config.server.git.searchPaths=config
spring.cloud.config.label=master
spring.cloud.config.server.git.username=your username
spring.cloud.config.server.git.password=your password

```
对于配置文件，我们作一下解释：

- spring.cloud.config.server.git.uri：配置git仓库地址
- spring.cloud.config.server.git.searchPaths：配置仓库路径
- spring.cloud.config.label：配置仓库的分支
- spring.cloud.config.server.git.username：访问git仓库的用户名
- spring.cloud.config.server.git.password：访问git仓库的用户密码

如果git仓库为公开仓库，可以不填写用户名和密码，如果是私有仓库需要填写，本例子是公开仓库，放心使用。

**4、web测试**

启动程序，使用Postman工具或浏览器访问：http://localhost:8888/conf/dev

![](https://ws2.sinaimg.cn/large/006tKfTcgy1fqtx0l97rwj315i0i0ju0.jpg)

证明配置服务中心可以从远程程序获取配置信息。我把github上文件conf-dev.properties内容更改为：jessehzx.habit=coding dev update，再次访问`http://localhost:8888/conf/dev`会发现结果为下图：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1fqtx6tj0grj315a0i0dig.jpg)

说明Server端会自动读取最新提交的配置文件内容。

http请求地址和资源文件映射如下:

```
/{application}/{profile}[/{label}]
/{application}-{profile}.yml
/{label}/{application}-{profile}.yml
/{application}-{profile}.properties
/{label}/{application}-{profile}.properties

```
以config-dev.properties为例子，它的application是config，profile是dev。client会根据填写的参数来选择读取对应的配置。

>**完整代码请点我查阅：[https://github.com/jessehzx/config-server](https://github.com/jessehzx/config-server)**

### 构建一个config client

**1、新建工程，添加依赖**

重新创建一个spring boot项目，取名为config-client，其pom.xml文件类似config-server项目，多加两个依赖，方便web测试，如下：

```
<dependencies>
	<dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-starter-config</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-web</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-test</artifactId>
		<scope>test</scope>
	</dependency>
</dependencies>
```

**2、配置文件**

需要配置两个配置文件，application.properties和bootstrap.properties。

配置文件bootstrap.properties：

```
spring.cloud.config.name=config
spring.cloud.config.profile=dev
spring.cloud.config.uri=http://localhost:8888/
spring.cloud.config.label=master
```
配置文件application.properties：

```
spring.application.name=config-client
server.port=8889
```

- spring.application.name：对应{application}部分
- spring.cloud.config.profile：对应{profile}部分
- spring.cloud.config.uri：配置中心的具体地址
- spring.cloud.config.label：对应git的分支。如果配置中心使用的是本地存储，则该参数无用
- spring.cloud.config.discovery.service-id：指定配置中心的service-id，便于扩展为高可用配置集群。

> 注意：上面这些与spring-cloud相关的属性必须配置在bootstrap.properties中，config部分内容才能被正确加载。因为config的相关配置会先于application.properties，而bootstrap.properties的加载也是先于application.properties。

**3、启动类**

程序的入口类，写一个API接口"/get_habit"，返回从配置中心读取的变量jessehzx.habit的值，代码如下：

```
@SpringBootApplication
@RestController
public class ConfigClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigClientApplication.class, args);
    }

    @Value("${jessehzx.habit}")
    String habit;
    @RequestMapping(value = "/get_habit")
    public String hi(){
        return habit;
    }
}
```

**4、web测试**

使用Postman工具或打开浏览器访问：http://localhost:8889/get_habit，网页显示：

![](https://ws2.sinaimg.cn/large/006tKfTcgy1fqtxf3y9thj30uc04ojrt.jpg)

这就说明，config-client从config-server获取了jessehzx.habit的属性，而config-server是从git仓库读取的。

**5、刷新**

我们再这个基础上进行一些小实验：手动修改config-dev.properties中配置信息为：jessehzx.habit=coding dev update1提交到github，再次在浏览器访问http://localhost:8889/get_habit，返回：jessehzx.habit:coding dev update，说明获取的信息还是旧的，这是为什么呢？因为spring boot项目只有在启动的时候才会获取配置文件的值，修改github信息后，client端并没有再次去获取，所以导致这个问题。如何去解决这个问题呢？

很简单，只需要加上@RefreshScope注解就可以了。使用该注解的类，会在接到SpringCloud配置中心配置刷新的时候，自动将新的配置更新到该类对应的字段中。

主类就长这样子：

```
@RefreshScope
@SpringBootApplication
@RestController
public class ConfigClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigClientApplication.class, args);
    }

    @Value("${jessehzx.habit}")
    String habit;
    @RequestMapping(value = "/get_habit")
    public String hi(){
        return habit;
    }
}
```
我们在访问一次看看，就发现已经将github上更新后的配置内容获取到客户端了：
![](https://ws4.sinaimg.cn/large/006tKfTcgy1fqtxquomjdj30n204awev.jpg)

>**完整代码请点我查阅：[https://github.com/jessehzx/config-client](https://github.com/jessehzx/config-client)**