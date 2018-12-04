---
layout:     post
title:      别扯那些没用的系列之: forEach循环
subtitle:   在你的项目中有多少是使用了这个特性？
date:       2018-11-13           
author:     jessehzx                
header-img: img/pexels-photo-1654495.jpeg
catalog: 	  true
tags:
    - forEach

---

> 版权声明：本文为 jessehzx 原创文章，支持转载，但必须在明确位置注明出处！！！

#### 序

写Java代码的程序员，集合的遍历是常有的事，用惯了for循环、while循环、do while循环，我们来点别的，JDK8 使用了新的forEach机制，结合streams，让你的代码看上去更加简洁、更加高端，便于后续的维护和阅读。好，不说了，"talk is cheap, show me the code"，我们直接上代码，秉承一贯以来的风格。skr~skr~

#### 一、对常用集合的遍历

JDK8中的forEach不仅可以对集合元素进行遍历，也能根据集合元素的值搞些事情，比如做判断，比如过滤。我们拿常用的List和Map来举例。

对Map集合的遍历：

```
/**
 * 对Map的遍历
 */
Map<String, Integer> map = Maps.newHashMap();
map.put("天猫双11", 1024);
map.put("京东双11", 1024);
// ①简写式
map.forEach((k, v) -> System.out.println("key:" + k + ", value:" + v));
// ②判断式
map.forEach((k, v) -> {
    System.out.println("key:" + k + ", value:" + v);
    if (StringUtils.contains(k, "京东")) {
        System.out.println("skr~");
    }
});
```

执行结果：

```
key:京东双11, value:1024
key:天猫双11, value:1024
key:京东双11, value:1024
skr~
key:天猫双11, value:1024
```

对List集合的遍历：

```
/**
 * 对List的遍历
 */
List<String> list = Lists.newArrayList();
list.add("买买买");
list.add("剁剁剁");
// ①简写式
list.forEach(item -> System.out.println(item));
// ②判断式
list.forEach(item -> {
    if (StringUtils.contains(item, "买")) {
       System.out.println("不如再用两个肾换个iPhone XS Max Pro Plus也无妨啊~");
    }
});
```
执行结果如下：

```
买买买
剁剁剁
不如再用两个肾换个iPhone XS Max Pro Plus也无妨啊~
```

#### 二、JDK8 中双冒号的使用

JDK8中有双冒号的用法，就是把方法当做参数传到stream内部，使stream的每个元素都传入到该方法里面执行一下。

比如，上面的`List<String>`的打印，我们可以这样写：

```
// JDK8 双冒号的用法
list.forEach(System.out::println);
```

执行结果也是一样一样的：

```
买买买
剁剁剁
```

在 JDK8 中，接口Iterable 8默认实现了forEach方法，调用了 JDK8 中增加的接口Consumer内的accept方法，执行传入的方法参数。
JDK源码如下：

```
/**
     * Performs the given action for each element of the {@code Iterable}
     * until all elements have been processed or the action throws an
     * exception.  Unless otherwise specified by the implementing class,
     * actions are performed in the order of iteration (if an iteration order
     * is specified).  Exceptions thrown by the action are relayed to the
     * caller.
     *
     * @implSpec
     * <p>The default implementation behaves as if:
     * <pre>{@code
     *     for (T t : this)
     *         action.accept(t);
     * }</pre>
     *
     * @param action The action to be performed for each element
     * @throws NullPointerException if the specified action is null
     * @since 1.8
     */
    default void forEach(Consumer<? super T> action) {
        Objects.requireNonNull(action);
        for (T t : this) {
            action.accept(t);
        }
    }
```

#### 三、对自定义类型的组装

这个用法我觉得是比较实用也比较常用的。我们先定义两个POJO，一个叫Track，是一个Entity，和我们的数据库表结构进行映射；另一个叫TrackVo，是一个Vo，在接口层返回给前端展示用。这里为了简化代码量，我们使用了lombok插件。好，先将它们定义出来：

Track.java
```
/**
 * @author huangzx
 * @date 2018/11/13
 */
@AllArgsConstructor
@Data
@Builder
public class Track {
    private Long id;
    private String name;
    private String anchor;
}
```
TrackVo.java
```
/**
 * @author huangzx
 * @date 2018/11/13
 */
@AllArgsConstructor
@Data
@Builder
public class TrackVo {
    private Long trackId;
    private String trackName;
}
```

经常遇到的场景就是：我通过一个Dao层将数据fetch出来，是一个`List<Track>`，但前端需要的是`List<TrackVo>`，但Track和TrackVo的字段又不一样。按照之前的做法，可能是直接用一个for循环或while循环将`List<Track>`遍历把里面的Entity赋值到TrackVo，你飞快地敲击键盘，伴随着屏幕的震动，十来行代码顿时被创造了出来，长舒一口气，大功告成！

殊不知，JDK8 自从引入新的forEach，结合streams，可以让这十来行代码浓缩为一行，实在是简练。来瞧一瞧：

```
/**
 * 对自定义类型的组装
 */
List<Track> trackList = Lists.newArrayList();
Track trackFirst = Track.builder().id(1L).name("我的理想").anchor("方清平").build();
Track trackSecond = Track.builder().id(2L).name("台上台下").anchor("方清平").build();
trackList.add(trackFirst);
trackList.add(trackSecond);

List<TrackVo> trackVoList = trackList.parallelStream().map(track -> TrackVo.builder().trackId(track.getId()).trackName(track.getName()).build()).collect(Collectors.toList());
System.out.println(JSON.toJSONString(trackVoList));
```

执行结果如下：

```
[{"trackId":1,"trackName":"我的理想"},{"trackId":2,"trackName":"台上台下"}]
```

似不似和你预期的结果一样？

#### 四、原理

ok，秉承程序员认识一件事物——“知其然必知其所以然”的原则。我们来分析一下forEach的实现原理。

首先，我们要了解一下上面用到的流 (streams) 概念，以及被动迭代器。

Java 8 最主要的新特性就是 Lambda 表达式以及与此相关的特性，如流 (streams)、方法引用 (method references)、功能接口 (functional interfaces)。正是因为这些新特性，我们能够使用被动迭代器而不是传统的主动迭代器，特别是 Iterable 接口提供了一个被动迭代器的缺省方法叫做 forEach()。缺省方法是 Java 8 的又一个新特性，是一个接口方法的缺省实现，在这种情况下，forEach() 方法实际上是用类似于 Java 5 这样的主动迭代器方式来实现的。

实现了 Iterable 接口的集合类 (如：所有列表 List、集合 map) 现在都有一个 forEach() 方法，这个方法接收一个功能接口参数，实际上传递给 forEach() 方法的参数是一个 Lambda 表达式。我们来编写一段使用 streams 的代码：

```
/**
 * @author huangzx
 * @date 2018/11/13
 */
public class StreamCountsTest {
    public static void main(String[] args) {
        List<String> words = Arrays.asList("natural", "flow", "of", "water", "narrower");
        long count = words.stream().filter(w -> w.length() > 5).count();
        System.out.println(count);
    }
}
```

上面所示代码使用 Java 8 方式编写代码实现流管道计算，统计字符长度超过5的单词的个数。列表 words 用于创建流，然后使用过滤器对数据集进行过滤，filter() 方法只过滤出单词的字符长度，该方法的参数是一个 Lambda 表达式。最后，流的 count() 方法作为最终操作，得到应用结果。

我们再对`自定义类型的组装`那句代码作个解析，如下：

![图解streams](https://ws1.sinaimg.cn/large/006tNbRwgy1fx6jy0lqd3j31kw06376y.jpg)

中间操作除了 filter() 之外，还有 distinct()、sorted()、map() 等，一般是对数据集的整理，返回值一般也是数据集。我们可以大致浏览一下它有哪些方法，如下：

![Streams提供的方法](https://ws4.sinaimg.cn/large/006tNbRwgy1fx6i39q2x1j314y17yqig.jpg)

总的来说，Stream 遵循 "做什么，而不是怎么去做" 的原则。在我们的示例中，描述了需要做什么，比如获得长单词并对它们的个数进行统计。我们没有指定按照什么顺序，或者在哪个线程中做。相反，循环在一开始就需要指定如何进行计算。

#### 五、为什么要用它？

网上许多人说：JDK8 的 Lambda 表达式的性能不如传统书写方式的性能。那为何还要出现呢？JDK的新api和新语法有时并不是为了性能而去做极致优化的。从理论上来说，面向对象编程，性能相对面向过程肯定是降低的，但是可维护性或清晰度有了很大的提升。

所以一个特性用与不用，取决于你关注什么，当公司给你3个月时间去做功能实现的时候，显然不会有人花1个月去做性能优化，这时候更清晰合理的代码就很重要了，大多数时候性能问题不是来自于算法和api的平庸表现，而是出自各种系统的bug。

#### 六、总结

总结一下上面讲了什么？首先是对于常见集合我们怎么用forEach去操作，并且介绍了双冒号的用法；然后基于一个已存在的集合怎么利用它产生一个新的集合，以封装成我们想要的数据；最后介绍了它的实现原理并阐释为何要用它的的原因。好了，下课。。。（老师~再见~~）

