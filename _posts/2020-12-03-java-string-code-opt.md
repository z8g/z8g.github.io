---
layout: mypost
title: Java 字符串代码性能
categories: [代码性能]
---

> 本文是一篇介绍代码性能的文章，由于笔者知识面有限，无法谈及架构设计上的性能优化，只能描述一些`代码层次`的优化方法。

[TOC]

谈到性能优化，笔者认为没有别的捷径，唯一的办法就是**测试，修改，再测试**。这里笔者使用 JMH(Java Microbenchmark Harness)来测试代码的性能，它是一款微基准测试工具(`org.openjdk.jmh`，通过 Maven 引入即可)。


字符串(String)，即字符数组(char[])，只要有可读数据的传输，就有字符串的身影；虽然还有很多可优化的例子可以提及，但是本文只针对 **Java 字符串操作方法间的差异** 进行介绍。


同时，为了解释清楚产生性能差异的原因，文中不得不贯穿一些 Java 前端编译优化(javac)和后端编译优化(JIT)的技术点，为了避免内容过泛，也无法具体地叙述。


## 1 JMH 主要参数的含义


本节将通过一个demo，向读者介绍 JMH 的常见参数。


以下是 3 个用于字符串格式化(format)操作的方法，读者先根据经验判断一下哪个方法最快。




> 字符串格式化是比较常用的字符串操作，例如日志等，下面三个方法分别基于String#format，StringBuilder#append，MessageFormat#format实现


```java
@Benchmark
public String byStringFormat() {
    return String.format("a: %s, b: %s, c: %s", a, b, c);
}

@Benchmark
public String byStringBuilder() {
    return "a: " + a + ", b: " + b + ", c: " + c;
}

@Benchmark
public String byMessageFormat() {
    return MessageFormat.format("a: {0}, b: {1}, c: {2}", a, b, c);
}
```


并且加入了`对照组`，empty 方法会直接返回一个新的 String 实例：


```java
@Benchmark
public String empty() {
    return new String("a: 1234, b: 56.78, c: abcd");
}
```

----
以下是 JMH 的运行结果，结果表示：
- 共测试了 4 个方法
- Mode=avgt，表示运行模式为平均运行时间(还可以设置为事务数)
- Cnt表示迭代10次(默认每次迭代运行1秒)
- Score和Error分别表示分数和误差，单位是Units，即纳秒/每操作
- 由于Mode=avgt，因此 Score 值越低，表示性能越好

Benchmark       |                    Mode | Cnt  |   Score & Error | Units
---- | ---- | ---- | ----: | ----
StringFormatMethod.empty          |  avgt |  10 |    9.716 ±  0.256 | ns/op
StringFormatMethod.byStringBuilder | avgt |  10 |  171.630 ±  2.517 | ns/op
StringFormatMethod.byStringFormat |  avgt |  10 | 1474.086 ± 36.624 | ns/op
StringFormatMethod.byMessageFormat | avgt |  10 | 2946.144 ± 71.755 | ns/op


显而易见，通过+号拼接实现的字符串格式化，性能远远高于 String#format 和 MessageFormat#format，原因如下：
- javac 在处理+号拼接的操作时，new 出一个StringBuilder实例，对每个+号操作，依次调用其 StringBuilder#append 方法，连接各个 String 实例，最后调用 StringBuilder#toString 方法返回结果
- String#format 底层使用正则表达式，虽然已经提前编译好了 Pattern ，但是模式匹配时，仍然引入了相当多的指令操作
- MessageFormat#format 底层虽然通过有限状态机(说直白些，就是通过for和if)优化了解析模板和渲染结果的过程，但是需要考虑的数据类型太多，还是不可避免地引入了许多耗时操作


因此在没有特殊的格式化需求(更具体地说，只有拼接字符串的需求)，直接使用+号即可，例如：



```java
Log.d("a: " + a + ", b: " + b + ", c: " + c);
```


----


本节最后，笔者将本次 JMH 输出结果的前几行拿到最后来介绍：
- 前4行分别表示 JMH 版本号，Java虚拟机版本、目录、参数
- Warmup 表示预热时间，Measurement 表示方法运行时间；即10次迭代(iterations)，每次1秒；测试结果都是 Measurement 中的
- Benchmark mode为平均运行时间，以及计算单位。
- Threads 为1，将同步执行 iterations


```bash
# JMH version: 1.19
# VM version: JDK 1.8.0_41, VM 25.40-b25
# VM invoker: /home/zhaoxuyang03/bin/jdk/java-se-8u41-ri/jre/bin/java
# VM options: -javaagent:/home/zhaoxuyang03/bin/idea/lib/idea_rt.jar=35083:/home/zhaoxuyang03/bin/idea/bin -Dfile.encoding=UTF-8
# Warmup: 10 iterations, 1 s each
# Measurement: 10 iterations, 1 s each
# Timeout: 10 min per iteration
# Threads: 1 thread, will synchronize iterations
# Benchmark mode: Average time, time/op
# Benchmark: net.zhaoxuyang.jmh.StringFormatMethod.byMessageFormat
```

## 2 尽可能地减少内存申请操作

### 2.1 字符串遍历操作中，toCharArray 性能慢于 charAt

以下有两种遍历String的方法，toCharArray方法会通过String#toCharArray方法返回一个新的char[]实例，而charAt会通过str.charAt来遍历String中的每个元素(charAt方法中含有try-catch语句)：


```java
/** JMH会为value中的每一项生成一组测试 */
@Param(value = {"short", "This is a long sentence..........................."})
String str;

/**
 * 通过{@link String#toCharArray}方法生成char数组后，遍历字符串
 */
@Benchmark
public int toCharArray() {
    char[] charArray = str.toCharArray();
    int res = 0;
    for (int i = 0; i < charArray.length; i++) {
        res += charArray[i];
    }
    return res;
}

/**
 * 在循环中通过调用charAt方法，遍历字符串
 */
@Benchmark
public int charAt() {
    int res = 0;
    for (int i = 0; i < str.length(); i++) {
        res += str.charAt(i);
    }
    return res;
}

```


----


以下是基准测试结果：
- toCharArray 方法的性能明显地比 charAt 方法的差，主要体现在 `str.toCharArray` 开辟内存的耗时
- 虽然 charAt 中使用了 try-catch 包裹，但还是没有申请内存的操作来得耗时，因此在使用中，是否可以考虑预先创建一个临时的char[]数组，反复使用？


-----


Benchmark       |                                                              (str) | Mode | Cnt |  Score &  Error | Units
----|----|----|----|----:|----
ToCharArrayOrCharAt.charAt    |                                                short | avgt  | 10 |  8.118 ± 0.273 | ns/op
ToCharArrayOrCharAt.charAt     |  This is a long sentence........................... | avgt  | 10 | 23.122 ± 0.611 | ns/op
ToCharArrayOrCharAt.toCharArray |                                              short | avgt  | 10 | 15.477 ± 1.176 | ns/op
ToCharArrayOrCharAt.toCharArray | This is a long sentence........................... | avgt  | 10 | 49.854 ± 2.164 | ns/op

### 2.2 通过预缓存来减少实际操作中的内存申请

本节会举两个例子，一个是 android.text.TextUtils#sTemp 字段，一个是目前许多模板引擎对 Integer#toString 方法的优化。


----


第一个例子是 android.text.TextUtils#sTemp 这个字段，提供给 TextUtils 内部使用，其主要目的就是为了减少内存申请操作，原理如下：
- 每当方法体中需要申请 char[] 类型的局部变量时(例如 TextUtils#indexOf 方法)，会调用 obtain(len) 方法返回一个临时数组sTemp，长度不够才重新申请内存
- 操作完成后，再调用设置 recycle 方法设置回 sTemp




```java
/* package */ static char[] obtain(int len) {
    char[] buf;

    synchronized (sLock) {
        buf = sTemp;
        sTemp = null;
    }

    if (buf == null || buf.length < len)
        buf = ArrayUtils.newUnpaddedCharArray(len);

    return buf;
}

/* package */ static void recycle(char[] temp) {
    if (temp.length > 1000)
        return;

    synchronized (sLock) {
        sTemp = temp;
    }
}
```


----


第二个例子是一个常见的优化手段，由于许多模板引擎会有整型转字符串的需求，因此考虑到性能，在符合应用场景的前提下，会预先缓存 Integer#toString 的结果，实现方法如下：




```java
/**
 * 提供toString方法的缓存工具类
 */
public static class Util {

    /** 缓存范围为 [0, CACHE_SIZE) */
    private static final int CACHE_SIZE = 2048;
    /** 缓存内容 */
    private static final String[] INT_CACHE;

    /* 预先生成toString结果 */
    static {
        INT_CACHE = new String[CACHE_SIZE];
        for (int i = 0; i < INT_CACHE.length; i++) {
            INT_CACHE[i] = Integer.toString(i);
        }
    }

    /** 不可实例化 */
    private Util() {
    }

    /**
     * int转String，超出缓存范围则通过 {@link Integer#toString} 生成结果
     */
    public static String intToString(int i) {
        return (i >= 0 && i < CACHE_SIZE) ? INT_CACHE[i] : Integer.toString(i);
    }
}
```


也就是说当入参的范围为[0, 2048)时，不再执行 Integer#toString 方法，直接返回预先计算的结果。


对于以下的测试用例：


```java
/**
 * JMH会为value中的每一项生成一组测试
 */
@Param(value = {"100", "1000", "10000"})
int value;

/**
 * 无缓存的toString方法
 */
@Benchmark
public String nonCache() {
    return Integer.toString(value);
}

/**
 * 预缓存的toString方法
 */
@Benchmark
public String cache() {
    return Util.intToString(value);
}
```


其基准测试结果如下：
- 当入参(value) 为 100 或 1000 时，可以走缓存，性能是未缓存时的8~9倍
- 当入参(value) 为 10000 时，未走缓存，性能与未缓存时持平(由于字节码指令较多，必然稍逊于后者)


Benchmark      |    (value) | Mode | Cnt |  Score & Error | Units
----|----|----|----|----:|----
PreCache.cache     |    100 | avgt  | 10  | 4.303 ± 0.113 | ns/op
PreCache.cache     |   1000 | avgt  | 10 |  4.255 ± 0.098 | ns/op
PreCache.cache     |  10000 | avgt  | 10 | 36.420 ± 0.697 | ns/op
PreCache.nonCache  |    100 | avgt  | 10 | 31.996 ± 7.915 | ns/op
PreCache.nonCache  |   1000 | avgt  | 10 | 36.673 ± 7.452 | ns/op
PreCache.nonCache  |  10000 | avgt  | 10 | 36.419 ± 1.337 | ns/op


## 3 使用语法糖时清楚实际运行的代码


### 3.1 不要使用+=来拼接字符串

Java 语法中，没有操作符重载的概念，但是编译器会为对象之间使用`+`号、`+=`号进行处理，下面只介绍 String 相关的运算符操作：
- 通过`+`号连接的String(其他类型会通过 String#valueOf 方法转换成String)实例，运行时会创建一个StringBuilder实例，通过append方法连接各String，最后通过 StringBuilder#toString 方法返回一个新实例
- 通过`+`号连接的String常量，编译期间javac会直接将该表达式改为一个常量(`常量折叠`)。
- `a += b; a+=c; `操作会导致运行时创建一个StringBuilder对象连接a与b，再将toString结果赋值给a；再创建一个StringBuilder对象，连接a与c，再将toString结果赋值给a

以下是一个使用 `+=` 的bad case：


```java
String a = "a";
String b = "b";
String c = "c";

/**
 * 通过+号拼接字符串
 */
@Benchmark
public String plus() {
    String res = a + b + c;
    return res;
}

/**
 * 通过StringBuilder的append方法拼接字符串
 */
@Benchmark
public String byStringBuilder() {
    String res = new StringBuilder().append(a).append(b).append(c).toString();
    return res;
}

/**
 * 通过+=形式拼接字符串
 */
@Benchmark
public String plusEquals() {
    String res = a;
    a += b;
    a += c;
    return res;
}
```


其基准测试结果如下：
- plusEquals 方法中使用了+=符号连接字符，虽然只进行了3个字符串实例的连接，但是其性能已经远远低于byStringBuilder和plus —— 如果放在一个长循环中使用，将造成更加严重的性能损耗
- 另一方面，可以看出 byStringBuilder 方法与 plus 方法性能相当 —— 其实两个方法的字节码完全一样(`javap -c *.class`)

Benchmark       | Mode | Cnt |  Score & Error | Units
----|----|----|----:|----
PlusOperator.byStringBuilder |  avgt  | 10  |    27.065 ±     1.003 | ns/op
PlusOperator.plus           |  avgt  | 10   |   27.178 ±     0.854 | ns/op
PlusOperator.plusEquals     |  avgt |  10 | 318331.676 ± 68204.792 | ns/op




### 3.2 switch(String) 的代替方案

Java 7 中，switch块里添加了对String类型的支持，例如：


```java
/** 键 */
String key = "code_1";

/**
 * 通过switch(String)语法糖来选择
 *
 * @return {@link #key} 的匹配结果
 */
@Benchmark
public int bySwitch() {
    String key = this.key;
    switch (key) {
        case "code_0":
            return 0;
        case "code_1":
            return 1;
        case "code_2":
            return 2;
        default:
            return -1;
    }
}
```


javac 解语法糖后，变成以下的等价形式：


```java
/**
 * switch 解语法糖后的等价形式：通过 {@link String#hashCode} 与 {@link String#equals} 来选择
 *
 * @return {@link #key} 的匹配结果
 */
@Benchmark
public int bySwitchByteCode() {
    String key = this.key;
    int hashCode = key.hashCode();
    switch (hashCode) {
        case -1355091362: // "code_0".hashCode()
            if ("code_0".equals(key)) {
                return 0;
            }
        case -1355091361: // "code_1".hashCode()
            if ("code_1".equals(key)) {
                return 1;
            }
        case -1355091360: // "code_2".hashCode()
            if ("code_2".equals(key)) {
                return 2;
            }
        default:
            return -1;
    }
}
```


通过 if 语句块的等价实现如下：


```java
/**
 * 通过 if 语句块的等价实现
 *
 * @return {@link #key} 的匹配结果
 */
@Benchmark
public int byIfEquals() {
    String key = this.key;
    if ("code_0".equals(key)) {
        return 0;
    } else if ("code_1".equals(key)) {
        return 1;
    } else if ("code_2".equals(key)) {
        return 2;
    } else {
        return -1;
    }
}
```


基于上述原理，可以预先计算"code_0"、"code_1"、"code_2"的hashCode，来实现代码性能的提升，实现如下：


```java
/** 预先计算好的"code_0"的hashCode */
final int PRE_HASH_CODE_0 = "code_0".hashCode();
/** 预先计算好的"code_1"的hashCode */
final int PRE_HASH_CODE_1 = "code_1".hashCode();
/** 预先计算好的"code_2"的hashCode */
final int PRE_HASH_CODE_2 = "code_2".hashCode();

/**
 * 通过预先计算 {@link String#hashCode()} 来选择
 *
 * @return {@link #key} 的匹配结果
 */
@Benchmark
public int byIfPreCache() {
    String key = this.key;

    int hashCode = key.hashCode();
    if (hashCode == PRE_HASH_CODE_0 && "code_0".equals(key)) {
        return 0;
    } else if (hashCode == PRE_HASH_CODE_1 && "code_1".equals(key)) {
        return 1;
    } else if (hashCode == PRE_HASH_CODE_2 && "code_2".equals(key)) {
        return 2;
    } else {
        return -1;
    }
}
```



以下是基准测试结果：
- bySwitchByteCode 比 bySwitch 稍快，是因为 javac 在编译期提前算好了字符串常量的hashCode(因为switch语句块中只能case常量)，最后输出了byteCode
- byIfEquals 方法直接通过 equals 方法进行选择，将直接进入 String#equals 方法中；而 byIfPreCache 中会先比较hashCode，避免了直接进入 String#equals 方法
- 另外，byIfPreCache 使用了提前计算好的 "code_1", "code_2","code_3" 的 hashCode，不用再在方法中重复计算，相比最初的 bySwitch 方法，提升了 33% 的性能


Benchmark    | Mode | Cnt |  Score & Error | Units
----|----|----|----:|----
SwitchString.byIfEquals   |     avgt   |10 | 8.966 ± 0.172 | ns/op
SwitchString.byIfPreCache  |    avgt  | 10 | 4.842 ± 0.169 | ns/op
SwitchString.bySwitch       |   avgt |  10 | 6.436 ± 0.142 | ns/op
SwitchString.bySwitchByteCode | avgt |  10 | 6.108 ± 0.092 | ns/op


## 4 对 StringBuilder 的补充说明


### 4.1 链式调用 StringBuidler#append 方法性能更好


本节主要建议(仅是一个建议)在方法内提前计算好局部变量的值，拼接字符串时，通过`append(str1).append(str2).append(str3).toStirng();` 的形式一次性输出结果。




以下是测试用例，empty 方法为对照组，chainAppend 方法为链式调用，nonChainAppend 方法为非链式调用的形式。


```java
String a = "a";
int b = 10;
char c = 'c';
boolean d = false;

/**
 * 对照组
 */
@Benchmark
public String empty() {
    StringBuilder sb = new StringBuilder();
    return sb.toString();
}

/**
 * 链式调用append方法
 */
@Benchmark
public String chainAppend() {
    StringBuilder sb = new StringBuilder();
    sb.append(a).append(b).append(c).append(d);
    return sb.toString();
}

/**
 * 非链式调用append方法
 */
@Benchmark
public String nonChainAppend() {
    StringBuilder sb = new StringBuilder();
    sb.append(a);
    sb.append(b);
    sb.append(c);
    sb.append(d);
    return sb.toString();
}
```


以下是基准测试结果：
- 结果表明，链式调用的性能略高于非链式调用
- 原因是这种非链式操作的append，每次都会从操作数栈中弹出，再从局部变量中装载引用类型值入栈


Benchmark       | Mode | Cnt |  Score & Error | Units
----|----|----|----:|----
AppendMode.empty          | avgt  | 10 | 18.203 ±  2.581 | ns/op
AppendMode.chainAppend   |  avgt  | 10 | 51.673 ±  8.233 | ns/op
AppendMode.nonChainAppend  |avgt |  10 | 59.500 ± 18.410 | ns/op




### 4.2 StringBuffer 不比 StringBuidler 慢多少


StringBuffer 是线程安全的，而 StringBuilder 非线程安全；既然前者通过 synchronized 修饰了方法，性能必然没StringBuilder好，但是其实没有差多少。


以下 case 会分别对 StringBuffer 和 StringBuilder 的实例进行十、百、万、百万次字符串拼接操作：


```java
@Param(value = {"10", "100", "10000", "1000000"})
int size;

@Benchmark
public String builder() {
    StringBuilder sb = new StringBuilder();
    for (int i = 0; i < size; i++) {
        sb.append("name").append(i).append('\n');
    }
    return sb.toString();
}

@Benchmark
public String buffer() {
    StringBuffer sb = new StringBuffer();
    for (int i = 0; i < size; i++) {
        sb.append("name").append(i).append('\n');
    }
    return sb.toString();
}
```




以下是基准测试结果：
- 结果表明在十至百万次拼接中，StringBuffer的性能相对于StringBuilder性能，只低了1% ~ 6%
- StringBuffer 跟 StringBuilder和相比性能并不差多少，得益于JIT C2阶段的逃逸分析和锁消除（对象只在方法内部使用，可以消除synchronized）
	- 逃逸分析：-XX:+DoEscapeAnalysis
	- 锁消除：-XX:+EliminateLocks
- 而实际上，方法内部局部变量以及方法参数是[线程私有](#)的，即不存在线程安全问题，此时编译器会直接提示开发者使用StringBuilder替换StringBuffer

Benchmark      |(size) | Mode | Cnt |  Score & Error | Units
----|----:|----|----|----:|----
StringBuilderBuffer.buffer    |    10 | avgt |  10  |     231.920 ±       5.211 | ns/op
StringBuilderBuffer.buffer    |   100 | avgt  | 10 |     3655.676 ±      97.173 | ns/op
StringBuilderBuffer.buffer  |   10000 | avgt  | 10 |   531097.767 ±   19279.096 | ns/op
StringBuilderBuffer.buffer  | 1000000 | avgt |  10 | 74592493.486 ± 1504365.581 | ns/op
StringBuilderBuffer.builder   |    10 | avgt |  10 |      228.170 ±       7.743 | ns/op
StringBuilderBuffer.builder   |   100 | avgt |  10 |     3275.142 ±     173.263 | ns/op
StringBuilderBuffer.builder  |  10000 | avgt |  10 |   492880.005 ±    7956.828 | ns/op
StringBuilderBuffer.builder | 1000000 | avgt |  10 | 70098295.407 ± 1517437.435 | ns/op


## 附录


### 附录A JMH 配置信息


```java
@BenchmarkMode(Mode.AverageTime) // 使用模式为运行时间，默认是Mode.Throughput，表示吞吐量
@Warmup(iterations = 10, time = 1, timeUnit = TimeUnit.SECONDS) // 预热
@Measurement(iterations = 10, time = 1, timeUnit = TimeUnit.SECONDS) // 运行
@Threads(1) // 同时执行的线程数
@Fork(1) // 为每个方法启动一个进程
@OutputTimeUnit(TimeUnit.NANOSECONDS) // 统计结果的时间单元
@State(Scope.Benchmark) // 对象的生命周期
public class BenchmarkTest {
    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder()
                .include(MethodHandles.lookup().lookupClass().getSimpleName())
                .forks(1)
                .build();
        new Runner(opt).run();
    }
}
```