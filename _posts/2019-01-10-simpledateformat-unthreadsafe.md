
#### 引子

最近手头上的项目上了一个新功能，每天早上一到公司，就兴致勃勃地登上服务器去查看日志，“窥视”一下跑的正不正常。今天终于碰到“彩蛋”了：

``` Java
Invalid Date in Date Math String:'2187-02-31T16:00:00Z'
...
Invalid Date in Date Math String:'0001-09-31T16:00:00Z'
```

这是什么鬼？怎么会有这样的日期？一会穿越到一百年后，一会穿越到原始社会，我想问那时的2月和9月都有31号了么？

#### 场景

冷静~ 我们先来理一理业务场景：我这边调用S团队的服务，接口参数传了String类型的开始日期和结束日期，格式：yyyy-MM-dd。既然报了“Invalid Date ...”错误，那是不是服务方对它们进行解析时出了问题呢？登上对方的服务器看日志去，发现很多 NumberFormatException：

```java
2019-01-10 00:31:22 380 [com.xxx.xxx.xxx.xxx.util.DataTool]-[WARN] 2019-01-09 00:00:00 parse err
java.lang.NumberFormatException: For input string: ".109E2.109E2"
        at sun.misc.FloatingDecimal.readJavaFormatString(FloatingDecimal.java:2043)
        at sun.misc.FloatingDecimal.parseDouble(FloatingDecimal.java:110)
        at java.lang.Double.parseDouble(Double.java:538)
        at java.text.DigitList.getDouble(DigitList.java:169)
        at java.text.DecimalFormat.parse(DecimalFormat.java:2056)
        at java.text.SimpleDateFormat.subParse(SimpleDateFormat.java:2162)
        at java.text.SimpleDateFormat.parse(SimpleDateFormat.java:1514)
        at java.text.DateFormat.parse(DateFormat.java:364)
        at com.xxx.xxx.xxx.xxx.util.DataTool.CCTToUTC(DataTool.java:29)


2019-01-10 00:31:22 415 [com.xxx.xxx.xxx.xxx.util.DataTool]-[WARN] 2019-01-10 00:00:00 parse err
java.lang.NumberFormatException: For input string: ""
        at java.lang.NumberFormatException.forInputString(NumberFormatException.java:65)
        at java.lang.Long.parseLong(Long.java:601)
        at java.lang.Long.parseLong(Long.java:631)
        at java.text.DigitList.getLong(DigitList.java:195)
        at java.text.DecimalFormat.parse(DecimalFormat.java:2051)
        at java.text.SimpleDateFormat.subParse(SimpleDateFormat.java:1869)
        at java.text.SimpleDateFormat.parse(SimpleDateFormat.java:1514)
        at java.text.DateFormat.parse(DateFormat.java:364)
        at com.xxx.xxx.xxx.xxx.util.DataTool.CCTToUTC(DataTool.java:29)
```

嗯，"2019-01-09 00:00:00" 和 “2019-01-10 00:00:00” 是我传过来的参数值，对应开始日期和结束日期。这应该没什么问题。那检查一下 DataTool.java 类 CCTToUTC 这个方法的第29行：

```java
public class DataTool {
	
	private static Logger logger = Logger.getLogger(DataTool.class);
	
	private static SimpleDateFormat dateSdf = new SimpleDateFormat("yyyy-MM-dd");
	
	private static SimpleDateFormat timezoneSdf = new SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss'Z'");
	
	public static String CCTToUTC(String timeString) {
		try {
			Date date = dateSdf.parse(timeString); // 第29行
			Calendar calendar = Calendar.getInstance();
			Date tgtDate = new Date(date.getTime() - calendar.getTimeZone().getRawOffset());
			return timezoneSdf.format(tgtDate);
		} catch (Exception e) {
			logger.warn(timeString+" parse err", e);
			return timezoneSdf.format(new Date());
		}
	}
}
```

代码很简单，定义全局变量 SimpleDateFormat，在 CCTToUTC(String timeString) 中用它对传入的日期进行解析和格式化。但在第一行 parse 的时候就报错了并被捕获到，而后打印了一行 warn 日志，并返回了当前时间 format 后的时间字符串。这不是我们想要的结果。

我怀疑是不是我传入的时间有问题，于是在本类写了个 main 方法，简单 sout 打印调用该方法后的结果，尝试了几个不同的时间串，发现始终得不到上面那些令我“穿越”的日期。

难道是别人也同时调用了该服务该方法？那为何在我这边的服务器日志上打印出来了？不可能。

还是找找自身的问题吧，从我开始调用一步一来分析。。。咦？调用的时候，为了性能，我写了一行很简练的代码：

```java
ids.parallelStream().forEach(id -> invokeMethod(id));
```

哦，并行处理？-> 并发？-> 线程安全？-> parse？-> SimpleDateFormat类？

是不是找到点线索？如果要进一步真正找到“嫌疑人”，那就还原一下现场嘛。。

```java
package com.jessehuang.dateformat;

import java.text.ParseException;
import java.util.Date;

public class DateUtilTest {
    
    public static class TestSimpleDateFormatThreadSafe extends Thread {
        @Override
        public void run() {
            while(true) {
                try {
                    this.join(2000);
                } catch (InterruptedException e1) {
                    e1.printStackTrace();
                }
                try {
                    System.out.println(this.getName() + ":" + DateUtil.parse("2019-01-10 00:00:00"));
                } catch (ParseException e) {
                    e.printStackTrace();
                }
            }
        }    
    }
    
    public static void main(String[] args) {
        for(int i = 0; i < 3; i++){
            new TestSimpleDateFormatThreadSafe().start();
        }
    }
}
```

输出结果：

```java
Exception in thread "Thread-1" Exception in thread "Thread-0" java.lang.NumberFormatException: multiple points
	at sun.misc.FloatingDecimal.readJavaFormatString(FloatingDecimal.java:1890)
	at sun.misc.FloatingDecimal.parseDouble(FloatingDecimal.java:110)
	at java.lang.Double.parseDouble(Double.java:538)
	at java.text.DigitList.getDouble(DigitList.java:169)
	at java.text.DecimalFormat.parse(DecimalFormat.java:2089)
	at java.text.SimpleDateFormat.subParse(SimpleDateFormat.java:1869)
	at java.text.SimpleDateFormat.parse(SimpleDateFormat.java:1514)
	at java.text.DateFormat.parse(DateFormat.java:364)
	at com.jessehuang.SimpleDateFormatTest.parse(SimpleDateFormatTest.java:21)
	at com.jessehuang.SimpleDateFormatTest$TestSimpleDateFormatThreadSafe.run(SimpleDateFormatTest.java:34)
java.lang.NumberFormatException: multiple points
	at sun.misc.FloatingDecimal.readJavaFormatString(FloatingDecimal.java:1890)
	at sun.misc.FloatingDecimal.parseDouble(FloatingDecimal.java:110)
	at java.lang.Double.parseDouble(Double.java:538)
	at java.text.DigitList.getDouble(DigitList.java:169)
	at java.text.DecimalFormat.parse(DecimalFormat.java:2089)
	at java.text.SimpleDateFormat.subParse(SimpleDateFormat.java:1869)
	at java.text.SimpleDateFormat.parse(SimpleDateFormat.java:1514)
	at java.text.DateFormat.parse(DateFormat.java:364)
	at com.jessehuang.SimpleDateFormatTest.parse(SimpleDateFormatTest.java:21)
	at com.jessehuang.SimpleDateFormatTest$TestSimpleDateFormatThreadSafe.run(SimpleDateFormatTest.java:34)
Thread-2:Sat Jan 10 00:00:00 CST 2201
Thread-2:Thu Jan 10 00:00:00 CST 2019
Thread-2:Thu Jan 10 00:00:00 CST 2019
Thread-2:Thu Jan 10 00:00:00 CST 2019
```
看到了吗？2201这种年份出现了。Thread-1和Thread-0报java.lang.NumberFormatException: multiple points错误，直接挂死，没起来；Thread-2 虽然没有挂死，但输出的时间是有错误的，比如我们输入的时间是：2019-01-10 00:00:00 ，但会输出：Sat Jan 10 00:00:00 CST 2201 这样的令人“穿越”的日期。

是的，破案了，凶手就是你 —— SimpleDateFormat

#### 分析

SimpleDateFormat 是 Java 中一个相当常用的类，该类用于对日期字符串进行解析和格式化，但如果使用不当会导致非常微妙和难以调试的问题，因为它不是线程安全的，在多线程环境下调用 format() 和 parse() 方法很容易产生问题。就像上面我一旦使用 JDK8 的 parallelStream() 来遍历，它就不好使了。

“知其然，必知其所以然” 。我们来分析一下为什么会输出奇怪的“穿越”日期。

我们打开 Dash 来查阅一下 JDK 文档 对于 SimpleDateFormat 的描述：

![](https://ws2.sinaimg.cn/large/006tNc79gy1fz2m5uheg4j317u05adi2.jpg)

下面通过源码来看看为什么 SimpleDateFormat 和 DateFormat 类不是线程安全的真正原因：

SimpleDateFormat 继承自 DateFormat，在 DateFormat 中定义了一个 protected 属性的 Calendar 类对象：calendar。因为 Calendar 类牵扯到了时区与本地化，JDK 的实现中使用了成员变量来传递参数，这就造成在多线程的时候会出现错误。

在 format() 方法里，有这样一段代码：

```java
private StringBuffer format(Date date, StringBuffer toAppendTo, FieldDelegate delegate) {
        // Convert input date to time field list
        calendar.setTime(date);

    boolean useDateFormatSymbols = useDateFormatSymbols();

        for (int i = 0; i < compiledPattern.length; ) {
            int tag = compiledPattern[i] >>> 8;
        int count = compiledPattern[i++] & 0xff;
        if (count == 255) {
        count = compiledPattern[i++] << 16;
        count |= compiledPattern[i++];
        }

        switch (tag) {
        case TAG_QUOTE_ASCII_CHAR:
        toAppendTo.append((char)count);
        break;

        case TAG_QUOTE_CHARS:
        toAppendTo.append(compiledPattern, i, count);
        i += count;
        break;

        default:
                subFormat(tag, count, delegate, toAppendTo, useDateFormatSymbols);
        break;
        }
    }
        return toAppendTo;
    }
```

calendar.setTime(date) 这条语句改变了 calendar ，然后，calendar 还在 subFormat() 方法里被用到，而这就是引发问题的根源。想象一下，在一个多线程环境下，有两个线程持有了同一个SimpleDateFormat 的实例，分别调用format方法：

- 线程1调用 format 方法，改变了 calendar 这个字段。
- 中断来了。
- 线程2开始执行，它也改变了 calendar。
- 又中断了。
- 线程1回来了，此时，calendar 已然不是它所设的值，而是走上了线程2设计的道路。如果多个线程同时争抢 calendar 对象，则会出现各种问题。比如时间不对，线程挂死等等。

分析一下 format() 的实现，我们不难发现，用到成员变量 calendar，唯一的好处，就是在调用 subFormat() 时，少了一个参数，却带来了这许多的问题。其实，只要在这里用一个局部变量，一路传递下去，所有问题都将迎刃而解。

#### 解决方案

方法一：

```java
public class DataTool {

    private static Logger logger = Logger.getLogger(DataTool.class);

    public static String CCTToUTC(String timeString) {
        try {
            Date date = getDateSdf().parse(timeString);
            Calendar calendar = Calendar.getInstance();
            Date tgtDate = new Date(date.getTime() - calendar.getTimeZone().getRawOffset());
            return getTimeZoneSdf().format(tgtDate);
        } catch (Exception e) {
            logger.warn(timeString + " parse err", e);
            return getTimeZoneSdf().format(new Date());
        }
    }

    private static SimpleDateFormat getTimeZoneSdf() {
        return new SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss'Z'");
    }

    private static SimpleDateFormat getDateSdf() {
        return new SimpleDateFormat("yyyy-MM-dd");
    }
}
```

在需要用到 SimpleDateFormat 的地方就新建一个实例。不管什么时候，将有线程安全问题的对象由共享变为局部私有都能避免多线程问题，不过也加重了创建对象的负担。在一般情况下，这样其实对性能影响也不是那么明显。

方法二：

```java
public class DateUtil {
    private static Logger logger = Logger.getLogger(DataTool.class);
    private static SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
    private static SimpleDateFormat timeZoneSdf = new SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss'Z'");

    public static String CCTToUTC(String timeString) {
        try {
            Date date = parse(timeString);
            Calendar calendar = Calendar.getInstance();
            Date tgtDate = new Date(date.getTime() - calendar.getTimeZone().getRawOffset());
            return formatDate(tgtDate);
        } catch (Exception e) {
            logger.warn(timeString + " parse err", e);
            return formatDate(new Date());
        }
    }

    private static Date parse(String strDate) throws ParseException {
        synchronized(sdf){
            return sdf.parse(strDate);
        }
    }

    private static String formatDate(Date date) throws ParseException {
        synchronized(timeZoneSdf){
            return sdf.format(date);
        }  
    }
}
```

当线程较多时，当一个线程调用该方法时，其他想要调用此方法的线程就要 block，多线程并发量大的时候会对性能有一定的影响。

方法三：

```java
public class DateUtil {
    private static Logger logger = Logger.getLogger(DataTool.class);
    private static ThreadLocal<DateFormat> threadLocal = new ThreadLocal<DateFormat>() {
        @Override
        protected DateFormat initialValue() {
            return new SimpleDateFormat("yyyy-MM-dd");
        }
    };

    private static ThreadLocal<DateFormat> threadLocal2 = new ThreadLocal<DateFormat>() {
        @Override
        protected DateFormat initialValue() {
            return new SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss'Z'");
        }
    };

    public static String CCTToUTC(String timeString) {
        try {
            Date date = parse(timeString);
            Calendar calendar = Calendar.getInstance();
            Date tgtDate = new Date(date.getTime() - calendar.getTimeZone().getRawOffset());
            return formatDate(tgtDate);
        } catch (Exception e) {
            logger.warn(timeString + " parse err", e);
            return formatDate(new Date());
        }
    }

    private static Date parse(String dateStr) throws ParseException {
        return threadLocal.get().parse(dateStr);
    }

    private static String formatDate(Date date) throws ParseException {
        return threadLocal2.get().format(date);
    }
}
```

方法四：抛弃JDK，使用其他类库中的时间格式化类：

- 使用 Apache commons 里的 FastDateFormat，“官宣”是既快又线程安全的 SimpleDateFormat, 可惜它只能对日期进行format(), 不能对日期串进行parse()
- 使用 Joda-Time 类库

其中，方法一和二，简单好用，推荐；方法三性能更优。

#### 总结

这也提醒我们在开发和设计系统的时候注意以下三点：

1、写工具类的时候，要对多线程调用情况下的后果在注释里进行明确说明

2、多线程环境下，对每一个共享变量都要注意其线程安全性

3、我们的类和方法在做设计的时候，要尽量设计成无状态的