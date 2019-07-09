---
layout:     post
title:      一次 lombok 默认值为 null 的排查实践
subtitle:   一次 lombok 默认值为 null 的排查实践
date:       2019-06-02             
author:     jessehzx                
header-img: img/post-bg-coffee.jpg
catalog: 	  true
tags:
    - 故障排查
        
---

> 微信公众号：JesseTalkJava

### 背景

这周的某个晚上，同事喊我过去看个问题，大概是这样的：为了满足新的业务需求，对于A、B两种不同的内容，在页面呈现上必须区分出两套规则，一套是用户可以进行修改和删除的，一套是用户只能查看的。

很容易想到一种做法就是：VO（View Object） 新增 Boolean 字段，对于 A、B 两种内容，组装 VO 的时候 A 的该字段设为 false，B 的该字段设为 true，通过 MVC 的 model 和 view 交互时对它作个判断，页面的区分渲染就可以实现了。

但是，在测试的时候，他发现一个奇怪的现象，就是这个新增的字段值竟然是 null，以致于根本无法作判断了。我们先看下 VO 的代码：

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class CommonVo {
    private Long id;
    private Integer status;
    @Builder.Default private Boolean readOnly = false; // 是否只读
}    
```

可以看到，在 CommonVO 中新增了名为 readOnly 的字段，并通过 lombok 注解 @Builder.Default 给它设置默认值为 false，而且这个 VO 类上也加了 lombok 注解，可以说一应俱全，乍一看没有任何问题，可为什么不行呢？

我们再看看组装 VO 的时候，是怎么实例化对象的：

```
CommonVO vo = new CommonVO();
```

哦，原来是用这种传统的实例化方式啊，那 new 一个无参构造函数，默认值就丢失了？

### 原因

我们直接编译一下 java 文件，看看 CommonVO 的 class 文件里面的无参构造函数是怎样的：

```
public CommonVo() {
}
```

无参构造函数体内竟然没有 this.readOnly = false; 这一行代码，那显而易见地，默认值根本就不会生效嘛。

然后再往下看，肯定是有一个 Builder 的构建器，没错在这里。为了划重点，我把 class 内容拷贝成文本后加了三个注解，代码如下：

```
public static class CommonVOBuilder {
    private Long id;
    private Integer status;
    private boolean readOnly$set;  // 重点关注①
    private Boolean readOnly;

    CommonVOBuilder() {
    }

    public CommonVO.CommonVOBuilder id(final Long id) {
        this.id = id;
        return this;
    }

    public CommonVO.CommonVOBuilder status(final Integer status) {
        this.status = status;
        return this;
    }

    public CommonVO.CommonVOBuilder readOnly(final Boolean readOnly) {
        this.readOnly = readOnly;
        this.readOnly$set = true;  // 重点关注②
        return this;
    }
	// 重点关注③
    public CommonVO build() {
        return new CommonVO(this.id, this.status, this.readOnly$set ? this.readOnly : CommonVO.$default$readOnly());
    }

    public String toString() {
        return "CommonVO.CommonVOBuilder(id=" + this.id + ", status=" + this.status + ", readOnly=" + this.readOnly + ")";
    }
}
```

可以看到，通过 lombok 编译后生成的 CommonVOBuilder 类会多出一个 `readOnly$set` 字段，这个字段的作用就是用来判断是否设置成默认值。譬如，在对象实例化的时候，如果设置了 readOnly 的值为 true，那么`readOnly$set` 就会被设为 true，调用 build() 方法之后，就会有一个三目运算符运算来决定它应该为 true 而不是默认值 false。这里的 `CommonVO.$default$readOnly()` 方法体内就一行代码，返回默认值：

```
private static Boolean $default$readOnly() {
    return false; // 这个false就是定义readOnly设置的默认值
}
```

如果我们从 POJO 的定义加上 lombok 注解，到对象实例化都用 lombok 的同一套风格来行事，肯定就不会出这岔子。

### 解决方案

那就直接把 

```
CommonVO vo = new CommonVO();
```

改为

```
CommonVO vo = CommonVO().builder().build();
```

嗯，非常好！这是最规范的写法，lombok 官网也推荐。

但是现实情况是：由于历史原因，项目中好多处都是直接 new 的形式来创建的，如果都按这种方式改，一是怕改漏了，二是怕改出问题。

有没有一种折中的办法，默认值无论是通过 new 方式还是 Builder().build() 方式都能正常使用呢？

好吧，既然要这种骚操作，那就再来一波探索。

首先一个问题就是：为什么 lombok 的无参构造函数没有帮我们设置默认值？

我看了下项目的 pom.xml 里面 lombok 的 dependency 是这样的：

![](http://ww1.sinaimg.cn/large/006tNc79gy1g3m62ek5eij30me04i3z5.jpg)

没指定 version，再往父依赖找：

![](http://ww3.sinaimg.cn/large/006tNc79gy1g3m64cqk6cj311808676d.jpg)

原来依赖的是 spring boot，在这个 pox 文件往上翻找查到具体版本：

![](http://ww4.sinaimg.cn/large/006tNc79gy1g3m66dfrklj30zw034aax.jpg)

然后我搜索了一下 maven 仓库，目前最高的是 1.18.8，抱着尝试的心态直接指定 lombok 的 version 为最新版，编译：

```
package com.example.demo.mock;
import com.example.demo.vo.CommonVO;
/**
 * @author Jessehuang
 */
public class LombokDefaultValTest {
    public static void main(String[] args) {
        CommonVO vo = new CommonVO();
        System.out.println(vo);
        CommonVO vo2 = CommonVO.builder().build();
        System.out.println(vo2);
    }
}
```

结果：

```
CommonVO(id=null, status=null, readOnly=false)
CommonVO(id=null, status=null, readOnly=false)
```

OK，可行！然后出于好奇心，我想知道到底是哪个版本开始支持的。多尝试了几个，发现 1.18.2 这个版本修正了这个问题，如果不信你可以把 lombok 的依赖指定为 1.18.2，再编译看看 class 文件就知道了。

另外，lombok 的 GitHub 的 issues 在2017年就有人提出这个问题，一年之后才得以修正。

如果不给 lombok 指定版本，还是依赖 spring boot 帮你指定，那必须把 spring boot 升级到 v2.1.0.M2 版本及以上才行，你可以在 spring boot 的 GitHub Releases 发版流水线的 Dependency upgrades 看到这个升级。

### 总结

总而言之，我们通过升级 lombok 版本的方式解决了默认值为 null 的问题。

其实 lombok 在帮我们减少 POJO 冗余编码的同时，也给我们带来了一些困扰。比如首字母小写第二个字母大写的命名方式就会造成 Jackson 失败问题、低版本默认值通过 new 方式初始化会为 null 的问题。

建议在编码的时候，不要交叉着使用上面两种实例化方式，这个必须要在团队中达成共识。