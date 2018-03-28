---
layout:     post
title:      深入理解Java集合之迭代器Iterator 
subtitle:   Iterator底层原理剖析，以及在日常开发过程中使用它易遇到的坑。
date:       2018-03-19             
author:     jessehzx                
header-img: img/post-bg-2015.jpg
catalog: 	  true
tags:
    - Java
        
---

>版权声明：本文为 jessehzx 原创文章，支持转载，但必须在明确位置注明出处！！！


### 开篇

相信大家在平时的Java开发中，对集合的遍历操作会使用相当频繁。比如，对ArrayList去重操作、在LinkedList中查询是否包含某个元素。都会涉及到对这个集合中元素的遍历，要么取出元素进行判断，要么进行删除，要么进行修改等等，那么这种取出并操作元素的动作用迭代器Iterator就再适合不过了。

### 用法
直接一上来就是用，先举个ArrayList去重的例子。代码示例一：

```
import java.util.ArrayList;
import java.util.Iterator;

/**
 * ArrayList去重
 */
public class ArrayListDemo {
    public static void main(String[] args) {
        ArrayList al = new ArrayList();
        al.add(1);
        al.add(2);
        al.add(1);
        al.add(3);
        System.out.println(removeDuplicates(al));
    }

    private static ArrayList removeDuplicates(ArrayList al) {
        ArrayList newAl = new ArrayList();
        for (Iterator it = al.iterator(); it.hasNext(); ) { // 循环体
            Object obj = it.next();
            if (!newAl.contains(obj)) {
                newAl.add(obj);
            }
        }
        return newAl;
    }
}
```
去重思路很简单：
1. 先定义一个新的ArrayList变量newAl，用于存储去重后的集合元素；
2. 使用Iterator遍历原list中的元素，如果该元素不包含于newAl，就add；
3. 返回newAl就是去重后的结果。
### 原理


> Iterator it = al.iterator();  // 定义迭代器，用于取出集合元素

代码一片段中定义迭代器，这是Java面向对象中多态的一种体现。这行代码定义了Iterator的一个子类对象。而子类是怎么建立出来的，不是通过new出来的，它是通过Collection中封装的方法获取出来的，它已经封装好了，JDK文档如下：

![Collection中的iterator方法](http://img.blog.csdn.net/20180312071853964?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvUm9ndWVGaXN0/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

> 那这个封装的方法到底是什么？为什么要这样来定义？

我们知道，对每个容器的操作都有存和取的方式，因为每个容器的数据结构不一样，所以存和取动作的实现方式是不一样的，比如顶层Collection接口中的add()方法，实现子类ArrayList的add()和LinkedList的add()实现方式是不一样的，前者的底层是数组，后者的底层是链表。

那既然数据结构都不一样，就导致它不能简单定义为一个方法来完成取出的功能，也就是说这种取出方式它不足于用父类Collection的一个方法来描述，它不像添加add()那么简单，它涉及到的往往不止一个动作，比如取出之前还要判断集合中有没有元素等等。那既然它包含不止一个功能，就把它封装成一个对象好了，让每个容器里面都持有这个对象，将其定义在集合内部（即内部类），这样就可以直接访问集合内的元素进行操作。

虽然每个容器取出的具体细节不一样，但是他们有共性的地方，比如都要判断还有没有元素可取，返回取出元素等等，我们把这些共性向上抽取对其封装成一个规则。这些内部类都必须符合这个规则。

这个规则被定义为Iterator。

> Iterator具体定义是什么样的？

Iterator是jdk1.2开始java.util下的一个接口，接口里面定义了两个抽象方法和两个default Method，如下：
![Iterator接口定义](http://img.blog.csdn.net/20180311220303135?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvUm9ndWVGaXN0/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
其中forEachRemaining(Consumer<? super E> action)是jdk1.8开始出现的方法，这里先不作赘述。

- boolean hasNext()：如果仍有元素可以迭代，则返回true。
- E next()：返回迭代的下一个元素。
- remove()：从迭代器指向的Collection中移除迭代器返回的最后一个元素（可选操作）。

> 如何获取集合的取出对象呢？

通过一个对外提供的方法——iterator()。我们可以看看定义在ArrayList里面的内部类实现，如下图：

![ArrayList内部类实现源码](http://img.blog.csdn.net/20180312104313917?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvUm9ndWVGaXN0/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
方法里面只有一行代码，直接返回new Itr();，来看看Itr是啥？如下图：
![Iterator内部类](http://img.blog.csdn.net/20180312105056541?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvUm9ndWVGaXN0/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

噢，原来内部类Itr实现了Iterator接口，覆写了hasNext()等抽象方法。各个方法体里面是对ArrayList下标的操作，根据下标的变化判断来取出元素。建议大家有时间可以去看看其他容器迭代器的源码实现，对于加深集合遍历的理解，和代码书写规范上都有很大帮助。


如果看了上面的阐述还不是很理解，那可以再举个简单的生活中的例子。

年轻人都玩过夹娃娃吧，那这个娃娃机就是一个集合，里面的娃娃就是集合中的元素，夹子在娃娃机里面，用来帮你取出娃娃，它就是迭代器Iterator。夹子有移动、张开、闭合等行为，这些就是对象中的方法。它被封装在内部，你只能通过娃娃机外边的操纵杆来控制它做一些事情，那这个操纵杆就是暴露在外边给你提供的取出方式。

### 陷阱

- 陷进一

代码示例二：

```
public class ArrayListDemo {
    public static void main(String[] args) {
        ArrayList al = new ArrayList();
        al.add(1);
        al.add(2);
        al.add(3);
        printList(al);
    }

    public static void printList(ArrayList al) {
        for (Iterator it = al.iterator(); it.hasNext(); ) {
            System.out.println(it.next() + "=====" + it.next()); // 两次next();
        }
    }
}
```

运行结果：
![迭代器循环中两次next()](http://img.blog.csdn.net/20180312124738882?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvUm9ndWVGaXN0/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

报这个异常是因为第二次判断时有元素3，但接下来取出只能取一次了，执行第二个it.next()就报错。所以，要养成一次next()之后就用hasNext()判断一次的使用习惯，取出一次判断一次接下来还没有元素可取才是稳妥的操作。

- 陷进二

代码示例三：

```
public class ArrayListDemo {
    public static void main(String[] args) {
        ArrayList al = new ArrayList();
        al.add(1);
        al.add(2);
        al.add(3);
        addItem(al);
    }

    public static void addItem(ArrayList al) {
        for (Iterator it = al.iterator(); it.hasNext();) {
            Object obj = it.next();
            if (obj.equals(2)) {
                al.add(5);  // 调用ArrayList的add()
            }
        }
    }
}
```

运行结果：
![迭代器循环中操作集合](http://img.blog.csdn.net/20180312125713910?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvUm9ndWVGaXN0/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

报这个异常是因为在用迭代器取出操作的同时，还用集合的方式操作元素，引发了并发访问的安全隐患。即你不能对一堆元素同时进行多种操作。

要么统一用集合的方法，要么统一用迭代器的方法。（其实迭代器有一个remove()方法，上面注释那一行改为al.remove()是可以的。）那有没有两者结合的方式呢？有，用listIterator()方法。代码示例四：

```
public static void addItem(ArrayList al) {
        for (ListIterator it = al.listIterator(); it.hasNext();) {
            Object obj = it.next();
            if (obj.equals(2)) {
                al.add(5);
            }
        }
    }
```
ListIterator接口继承自Iterator，但多提供了添加add()、删除remove()和修改set()等方法。

### 总结
- 编码习惯

大家有没有发现，在代码示例一片段中，有一行代码的写法稍微特别一点，就是我加了注释的那一行。
> for (Iterator it = al.iterator(); it.hasNext();) { // 循环体

我之所以这样写，因为一般情况下，做完一次循环遍历操作后，基本上不会在同一个代码块（函数）中再做第二次循环，那Iterator变量it是不是可以马上回收它节约内存？是的，把它定义在for循环里面成为局部变量就可以做到。一般大家在“市面上”或自己的习惯写法是这样的。代码示例五：
    
```
    Iterator it = al.iterator();
        while (it.hasNext()) {
            it.next();
    }
```

- 内部类的使用

容器是一类事物，容器内部还有一个事物，把它定义成内部类，这个内部类直接访问容器的内部成员，这就是集合中的内部类。而且这个内部类还实现了接口，对外提供规则，我们只需要面对规则就行了，你内部怎么实现跟我没关系，我只要规则就完事了。

> 本文首发于我的CSDN，链接：[*http://blog.csdn.net/roguefist/article/details/79526657*](http://blog.csdn.net/roguefist/article/details/79526657)