---
layout:     post
title:      力荐十个Github优质Java项目，你都看过了？
subtitle:   Github上搜索的奇技淫巧，你会了吗？
date:       2018-10-24           
author:     jessehzx                
header-img: img/post-bg-os-metro.jpg
catalog: 	  true
tags:
    - 学习方法论
        
---

> 版权声明：本文为 jessehzx 原创文章，支持转载，但必须在明确位置注明出处！！！

前天Github down了24小时，于昨天早上07:03分才恢复正常，心想参与急救的程序员应该整整一个白天一个黑夜都没合过眼吧。

昨晚登陆账号上来看看，然后在网上发现一个奇技淫巧，可以在Github上按搜索条件查询出项目的排行榜，比如你想获取一个点赞数大于3000的Java项目，可以在搜索框中输入`Java stars:>3000`：

![](https://ws4.sinaimg.cn/large/006tNbRwgy1fwib0isklmj31g605o75n.jpg)

然后在查询结果页面的右边Sort options选择Most starts，你要的排行榜就出炉了。

![](https://ws3.sinaimg.cn/large/006tNbRwgy1fwib10s824j315a0bswgs.jpg)

俗话说：一切创新都从模仿开始。只有吸天地之灵气汲日月之精华，才能有取之不尽、用之不竭的能量。猿们学习编程也是一样一样的，多看优质源码，多跟大神"对话"，方能提升自己的代码"格局"。于是我花了一些精力挑选出近几个月Github上最热门的10个Java项目供大家玩赏，罗列出来，自己感兴趣的，不妨check out学习一下！

① java-design-patterns（Star:36k）
> https://github.com/iluwatar/java-design-patterns

介绍：设计模式大家都不陌生，代理、工厂、观察者 ... 程序员可以在设计应用程序时使用它来解决复杂场景的问题，重用设计模式也有助于提高代码的可读性，使之看起来更加优雅和高端。

② Elasticsearch (Star:32k)
> https://github.com/elastic/elasticsearch

介绍：ElasticSearch是一个基于Lucene的搜索服务器。比如我们建一个网站，要添加搜索功能，希望要运行速度快，希望能有一个零配置和一个完全免费的搜索模式，希望能够简单地使用JSON通过HTTP来索引数据，希望搜索服务始终可用，希望能够从一台扩展到数百台，要实时搜索，要简单的多租户，还要建立一个基于云的解决方案，因此我们可以利用Elasticsearch来为我们排忧解难，去实现上述要求。

③ okhttp(Start:27k)
> https://github.com/square/okhttp

介绍：做Android的同学应该很熟悉。okhttp 适用于 Android 和 Java 应用程序，即使搞后端的也建议了解下，有助于对 HTTP 协议有更进一步的认识。

④ spring-boot (Star:26k)
> https://github.com/spring-projects/spring-boot

介绍：虽然Spring的组件代码是轻量级的，但它的配置却是重量级的（需要大量XML配置），不过 Spring Boot 让这一切成为了过去。关于 Spring Boot 的介绍，官方文档毋庸置疑是最权威最详尽的，但如果你觉得英文文档上手起来太慢，这里有一篇中文的，推荐给你：[《Spring Boot参考指南》](https://www.mangocd.com/gbook/spring-boot-reference-guide/index.html#)

⑤ guava (Star:25k)
> https://github.com/google/guava

介绍：Guava是一组核心库，包括新的集合类型（例如multimap和multiset），不可变集合，图形库，函数类型，内存缓存以及用于并发，I/O，散列，反射，字符串处理等等，应有尽有！

⑥ incubator-dubbo (Star:20k)
> https://github.com/apache/incubator-dubbo

介绍：Apache Dubbo（孵化）是阿里开源的一个基于Java的高性能开源RPC框架。

⑦ proxyee-down（Star:11k）
> https://github.com/proxyee-down-org/proxyee-down

介绍：百度云资源下载了解一下，亲测速度真心不错！一个http下载工具，基于http代理，支持多连接分块下载。

⑧ weixin-java-tools (Star:8.4k)
> https://github.com/Wechat-Group/weixin-java-tools

介绍：可能是目前最好最全的微信Java开发工具包，支持包括微信支付、开放平台、小程序、企业号和公众号等的开发。

⑨ apollo（Star:6.5k）
> https://github.com/ctripcorp/apollo

介绍：Apollo（阿波罗）是携程框架部门研发的分布式配置中心，能够集中化管理应用不同环境、不同集群的配置，配置修改后能够实时推送到应用端，并且具备规范的权限、流程治理等特性，适用于微服务配置管理场景。

⑩ jib（Star:3.4k）
> https://github.com/GoogleContainerTools/jib

介绍：Google 最近开源的一款 Java 工具，旨在让开发者更轻松地将 Java 应用程序容器化。在过去，容器化 Java 应用并非易事：你必须先编写 Dockerfile ，root 后运行 Docker 守护进程，等待构建完成，最后将镜像推送至远程注册表。而 Jib 能帮你处理应用打包到容器镜像过程中的所有步骤，它直接与 Maven 和 Gradle Java 开发环境集成，不需要你编写 Dockerfile 或安装 Docker ，只需将其作为插件添加到你的构建中，就可以立即将 Java 应用容器化。相关阅读：[《Google 正式开源 Jib ，帮助 Java 应用快速容器化》](https://www.oschina.net/news/97892/google-opensource-jib)

面对那么多牛逼的优质资源，是不是蠢蠢欲动？承认吧，因为“蠢”所以才要行动起来，学起来啊！
