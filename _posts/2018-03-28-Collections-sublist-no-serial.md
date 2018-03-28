---
layout:     post
title:      解决集合subList()报NotSerializableException问题
subtitle:   解决集合subList()报未序列化的Exception
date:       2018-03-28             
author:     jessehzx                
header-img: img/post-bg-os-metro.jpg
catalog: 	  true
tags:
    - Jave
        
---

>版权声明：本文为 jessehzx 原创文章，支持转载，但必须在明确位置注明出处！！！


### 序

今天应产品需求变更，对某个业务的代码做了一点小调整。其中用到了`ArrayList`的`subList()`方法，将切分后的子集再`set`进`memcached`。编译通过，但运行时，报了以下这个异常：

```
Caused by: java.io.NotSerializableException: java.util.RandomAccessSubList	at java.io.ObjectOutputStream.writeObject0(ObjectOutputStream.java:1164)	at java.io.ObjectOutputStream.defaultWriteFields(ObjectOutputStream.java:1518)	at java.io.ObjectOutputStream.writeSerialData(ObjectOutputStream.java:1483)	at java.io.ObjectOutputStream.writeOrdinaryObject(ObjectOutputStream.java:1400)	at java.io.ObjectOutputStream.writeObject0(ObjectOutputStream.java:1158)	at java.io.ObjectOutputStream.writeObject(ObjectOutputStream.java:330)	at java.util.ArrayList.writeObject(ArrayList.java:570)	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
```
也就是说：当我尝试着将一个`subList()`的子集，添加到`memcached`时遇到了奇怪的序列化问题。

### 原因
存在即合理，凡事肯定有它的缘由。既然是`set`进`memcached`报了`NotSerializableException`，顾名思义就是没有序列化嘛，那就看看`subList()`返回了啥呗，我`option`+`command`+点进去，这玩意还是个内部类：

```
private class SubList extends AbstractList<E> implements RandomAccess {}
```
而ArrayList类是实现了`Serializable`接口的，所以和`memcached`打交道的时候它行云流水：

```
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
```

那很显然了，问题就出在用`Collections.subList()`方法返回了不可序列化的`java.util.RandomAccessSubList`实例。

### 解决

将`subList()`子集重新放入`ArrayList`中。

```
new ArrayList<String>(list.subList(0, 3));
```

### 总结
基础知识永远是最重要的，只有基础扎实了，才能走的更远。

