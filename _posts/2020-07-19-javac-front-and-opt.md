---
layout: mypost
title: Java 前端编译与优化
categories: [Java基础]
---


## 1 概述
代表性的编译器产品:
- 前端编译器: JDK的Javac、Eclipse JDT中的增量式编译器(ECJ)
- 即时编译器: HotSpot虚拟机的C1、C2编译器，Graal编译器
- 提前编译器: JDK的Jaotc、GNU Compiler for the Java(GCJ)、Excelsior JET


> 即时编译器在运行期的优化，支撑了程序执行效率的不断提升；而前端编译器在编译期的优化过程，支撑着程序员的编码效率和语言使用者的幸福感的提高


## 2 Javac 编译器


Javac编译过程大致分为1个准备过程和3个处理过程:
- (1) 准备过程: 初始化插入式注解处理器
- (2) 解析与填充符号表过程:
	- 词法、语法分析: 将源代码的字符流转变为标记集合，构造出抽象语法树
	- 填充符号表: 产生符号地址和符号信息
- (3) 插入式注解处理器的注解处理过程
	- 如果产生新的符号，就必须回到之前的解析与填充符号表过程
- (4) 分析与字节码生成过程:
	- 标注检查: 对语法的静态信息进行检查
	- 数据流分析与控制流分析: 对程序动态运行过程进行检查
	- 解语法糖: 将简化代码编写的语法糖还原为原有的形式
	- 字节码生成: 将前面各个步骤所生成的信息转化成字节码

> javac 源码 `com.sun.tools.javac.main.JavaCompiler`

```java
initProcessAnnotations(processors); // 准备过程: 初始化插入式注解处理器
delegateCompiler = 
	processAnnotations(	// 过程2: 执行注解处理
		enterTrees( // 过程1.2 填充符号表
			stopIfError(CompileState.PARSE,
			parseFiles(sourceFileObjects) // 过程1.1 词法分析、语法分析
			)
		),
		classnames
	);

delegateCompiler.compile2(); // 过程3 分析及字节码生成
/*
case BY_TODO:
	while (!todo.isEmpty()) {
		// 过程3.4 生成字节码( 过程3.3 解语法糖( 过程3.2 数据流分析( 过程3.1 标注
		generate(desugar(flow(attribute(todo.remove()))));
	}
*/
```
### 2.1 解析与填充符号表
#### 2.1.1 词法、语法分析
- 词法分析过程由 `com.sun.tools.javac.parser.Scanenr` 将源码字符流转换成Token
- 语法分析过程由 `com.sun.tools.javac.tree.JCTree` 按照Token序列构造出抽象语法树(AST)
- 后续操作都是建立在AST上

#### 2.1.2 填充符号表
- 符号表(Symbol Table)是由一组符号地址和符号信息构成的数据结构，由 `com.sun.tools.javac.comp.Enter` 实现，输出一个待处理列表，包含每一个编译单元的AST的顶级节点以及 `package-info.java` 的顶级节点


### 2.2 注解处理器
- 插入式注解(Anntations)处理器工作时，可以读取、修改、添加AST中的任意元素，如果进行过修改操作，编译器将回到解析及填充符号表的过程，直到所有插入式注解处理器都没有再对AST进行修改，每次循环成为一个轮次(Round)
- 例如 `Lombok` 可以通过注解来实现 `getter/setter` 等方法
- 其初始化过程在 `initProcessAnnotations()` 方法，其执行过程在 `processAnnotations()` 方法，如果有新的注解处理器需要执行，则通过 `com.sun.tools.javac.processing.JavaProcessingEnvironment#doProcessing()` 方法生成一个新的 `JavaCompiler` 对象，对编译后续步骤进行处理


### 2.3 语义分析与字节码生成
语法分析的主要任务是对结构上正确的源程序进行上下文相关性质的检查，比如类型检查、控制流程检查、数据流检查等
- 例如重复定义变量，则不会通过检查
- IDE上红线标注的错误，绝大部分源于语法分析阶段的检查结果


#### 2.3.1 标注检查
标注检查由 `com.sun.tools.javac.comp.Attr` 和 `com.sun.tools.javac.comp.Check` 实现，要检查的内容包括:
- 变量使用前是否已被声明
- 变量与赋值之间的数据类型是否能够匹配
- ...
- 在标注检查中，会进行常量折叠(Constant Floding)的优化


> `常量折叠`优化的例子


```java
int a = 1 + 2; // 等价于int a = 3;
String str = "hello" + " " + "World"; // 等价于String str = "hello World";
```



#### 2.3.2 数据流分析与控制流分析
数据流分析和控制流分析由 `com.sun.tools.javac.comp.Flow` 实现，是对程序上下文逻辑更进一步的验证，可以检查:
- 程序局部变量在使用前是否有赋值
- 方法的每条路径是否有返回值
- 是否所有的受检异常都被正确处理
- ...


> 由于局部变量在常量池中没有 `CONSTANT_Fieldref_info` 的符号引用，所以 `final` 修饰的局部变量对运行时没有影响: 


```java
void method1(final int i) {
	final int j = 0;
}
void method2(int i) {
	int j = 0;
}
// 字节码反编译结果
/*
  void method1(int);
    Code:
       0: iconst_1
       1: istore_2
       2: return

  void method2(int);
    Code:
       0: iconst_1
       1: istore_2
       2: return
*/
```


#### 2.3.3 解语法糖
解语法糖的过程由 `com.sun.tools.javac.comp.TransTypes` 和 `com.sun.tools.javac.comp.Lower` 完成


#### 2.3.4 字节码生成
字节码(Byte Code)由 `com.sun.tools.javac.jvm.Gen` 完成，生成过程中不仅把AST和符号表转化成字节码指令写到磁盘，还进行了少量的代码添加和转换，主要内容包含:
- 实例构造器 `<init>()` 方法和类构造器 `<client>()` 方法在这个这个阶段被添加到AST中
- 把语句块(对于实例构造器是`{}`，对于类构造器是`static{}`)、变量初始化、调用父类的实例构造器等操作收敛到`<init>()` 和 `<client>()` 方法中
- 按照先执行父类的实例构造器，初始化变量，最后执行语句块的顺序进行(由 `Gen::nomalizeDefs()` 方法实现)
- 把字符串的加号操作替换成StirngBuffer(JDK 5以上为StringBuilder)的append操作等


完成了AST的遍历和调整之后，就会通过 `com.sun.tools.javac.jvm.ClassWriter#writeClass` 方法将填充好的符号表输出成字节码，生成Class文件，整个编译过程结束。


## 3 Java 语法糖


语法糖(Syntactic Sugar)是指在计算机语言中添加的某种语法，对语言的编译结果和功能并没有实际影响，但是却能更方便程序员使用该语言。通常来说使用语法糖能够减少代码量、增加程序的可读性，从而减少程序代码出错的机会
### 3.1 泛型
泛型的本质是参数化类型(Parameterized Type)或者参数多态化(Parametric Polymorphism)的应用，即可以将操作的数据类型指定为方法签名中的一种特殊参数


#### 3.1.1 实现方式
Java泛型的实现方式为类型擦除式形式(Type Erasure Generics)，只在源码中存在，编译后的字节码文件中，都被替换为原来的裸类型(Raw Type)。因此Java泛型有以下不被支持的用法:


> Java 中不支持的泛型用法


```java
public class A<E> {
	public void method(Object item) {
		if (item instanceof E) { // 无法对泛型进行实例判断
			// ...
		}
		E e = new E(); // 无法使用泛型创建对象
		E[] arr = new E[2]; // 无法使用泛型创建数组
	}
}
```


#### 3.1.2 历史背景


Martin Odersky教授在刚发布一年的Java上实现了函数式编程的三大特性，泛型、高阶函数和模式匹配，形成了Scala语言的前身Pizza语言，后来和Java团队建立了名为 `Generic Java` 的项目，目标是把Pizza语言的泛型移植到Java中，但是需要保证二进制向后兼容性(Binary Backwards Compatibility，明确写入《Java语言规范》中的对Java使用者的严肃承诺，譬如一个在JDK1.2中编译出来的Class文件，必须保证能够在JDK12乃至以后的版本中也能够正常运行)，意味着以前没有的限制不能突然间冒出来。


> `Java 5.0` 以前数组支持协变(Covariant)，集合类可以存入不同类型的元素，下面的代码可以被正常编译:


```java
Object[] array = new String[10];
array[0] = 10; // 编译成功，运行出错


ArrayList list = new ArrayList();
list.add(10); // 编译成功，运行成功
list.add("Hello");
```


因此设计者有两种选择:
- 平行地加一套泛型化版本的新类型，例如C#添加了 `System.Collections.Generic`，Java引入了`Vector` 和 `Hashtable`(Java曾经的尝试)
- 直接把所有的类型泛型化(Java后来的选择)，例如将ArrayList写为 `ArrayList<T>`，Java中使用类型擦除实现


#### 3.1.3 泛型擦除


以ArrayList为例，裸类型的实现有两种选择:
- 在运行期由JVM来自动地、真实地构造出 `ArrayList<Integer>` 这样的类型，并自动实现从 `ArrayList<Integer>` 派生自ArrayList的继承关系来满足裸类型的定义
- 直接在编译时把ArrayList<Integer>还原为ArrayList，在元素访问、修改时自动插入一些强制类型转换和检查指令


> 以下是一个展示`类型擦除`前后的例子:


```java
// 类型擦除前
Map<String, String> map = new HashMap<>();
map.put("Hello", "你好");
map.get("Hello");


// 类型擦除后
Map map = new HashMap();
map.put("Hello", "你好");
map.get((String)"Hello");
```

以下是类型擦除带来的问题:
- 由于不支持int、long与Object之间的强制转换，因此目前的Java不支持对原始类型(Primitive Types)的泛型，当时Java给出的解决方案是将泛型中的Integer等包装类自动转换成原始类型，这个决定导致了无数构造包装类和装箱、拆箱的开销，成为Java泛型慢的重要原因，也成为 [Valhalla](https://zhuanlan.zhihu.com/p/87960146) 项目要重点解决的问题之一

- 由于运行期无法获取到泛型类型的信息，因此一些代码会变得繁琐，例如写一个泛型版本的从List到数组的转换方法，由于不能从List中取得`T`，所以不得不新增一个参数传入数组的类型


> `List<T>` 到 `T[]` 的转换方法


```java
public static <T> T[] convert(List<T> list, Class<T> type) {
	T[] array = (T[]) Array.newInstance(type, list.size());
	// ...
}
```


### 3.2 自动装箱、拆箱与遍历循环

以下的例子包含了泛型、自动装箱、拆箱与遍历循环、变长参数5种语法糖，两个方法编译出来的字节码完全相同。


> 包含泛型、自动装箱、拆箱与遍历循环、变长参数5种语法糖的例子


```java
void before() {
    List<Integer> list = Arrays.asList(1, 2, 3);
    int sum = 0;
    for (int i : list) {
        sum += i;
    }
    System.out.println(sum);
}
void after() {
    List list = Arrays.asList(new Integer[]{
            Integer.valueOf(1),
            Integer.valueOf(2),
            Integer.valueOf(3)
    });
    int sum = 0;
    for (Iterator iter = list.iterator(); iter.hasNext(); ) {
        int i = ((Integer) iter.next()).intValue();
        sum += i;
    }
    System.out.println(sum);
}
```

避免对包装类使用运算符，下面是一些陷阱:

> 一个关于`自动装箱、拆箱`陷阱的例子


```java
Integer a = 1;
Integer b = 2;
Integer c = 3;
Integer d = 3;
Integer e = 128;
Integer f = 128;
Long g = 3L;
System.out.println(c == d); // true，Integer.valueOf(c) == Integer.valueOf(d)，比较的是Integer.IntegerCache.cache，地址相同
System.out.println(e == f); // false，Integer.valueOf(e) == Integer.valueOf(f)，比较的是new Integer()，地址不同
System.out.println(g.equals(a + b)); // false，Long.equals(Integer.valueOf(a.intValue() + b.intValue()))，Long#equals方法的instanceof比较中返回false
```

> 以下是 `Integer#valueOf` 和 `Long#equals` 的源码

```java
// java.lang.Integer
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
// java.lang.Long
public boolean equals(Object obj) {
    if (obj instanceof Long) {
        return value == ((Long)obj).longValue();
    }
    return false;
}
```

### 3.3 条件编译


以下是关于条件编译的示例，method0()、method1()、method2()三个方法编译生成的字节码完全相同，method3()方法中由于变量的缘故，不进行条件编译。


> 条件编译的例子


```java
void method0() {
    System.out.println(true);
}
void method1() {
    if (true) {
        System.out.println(true);
    } else {
        System.out.println(false);
    }
}
void method2() {
    if (1 + 1 == 2) {
        System.out.println(true);
    } else {
        System.out.println(false);
    }
}
void method3() { 
    int a = 1;// 变量不参与条件编译
    if (a + 1 == 2) {
        System.out.println(true);
    } else {
        System.out.println(false);
    }
}
```
## 4 插入式注解处理器

可以使用插入式注解处理器API来对Java编译子系统的行为施加影响


### 4.1 NameCheckProcessor: 名称检查插件


> 以下是名称检查插件的代码 `NameCheckProcessor.java`


```
import javax.annotation.processing.*;
import javax.lang.model.SourceVersion;
import javax.lang.model.element.Element;
import javax.lang.model.element.TypeElement;
import java.util.Set;

/**
 * 名称检查插件
 */
@SupportedAnnotationTypes("*") // 不限于特定的注解，检查任何代码
@SupportedSourceVersion(SourceVersion.RELEASE_6) // 能处理基于JDK 6的源码
public class NameCheckProcessor extends AbstractProcessor {

	/** 名称检查器 */
    private NameChecker nameChecker;

    /**
     * 初始化名称检查插件
     */
    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        nameChecker = new NameChecker(processingEnv);
    }

    /**
     * 对输入的语法树的各个节点进行名称检查
     */
    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        if (!roundEnv.processingOver()) {
            // 在当前轮次(round)中每一个根节点传入检查器进行检查
            for (Element element : roundEnv.getRootElements()) {
                nameChecker.checkNames(element);
            }
        }
        return false;
    }
}
```


### 4.2 NameChecker: 名称检查器


> 以下是名称检查器的代码 `NameChecker.java`


```java
import javax.annotation.processing.Messager;
import javax.annotation.processing.ProcessingEnvironment;
import javax.lang.model.element.*;
import javax.lang.model.util.ElementScanner6;
import javax.tools.Diagnostic;

import java.util.EnumSet;

import static javax.lang.model.element.ElementKind.*;
import static javax.lang.model.element.Modifier.*;

/**
 * 名称检查器
 */
public class NameChecker {
    
    /** 消息通知器 */
    private final Messager messager;
    /** 自定义的名称扫描器 */
    NameCheckerScanner nameCheckerScanner = new NameCheckerScanner();

    /**
     * 名称检查器的构造函数
     */
    public NameChecker(ProcessingEnvironment processingEnv) {
        this.messager = processingEnv.getMessager();
    }

    /**
     * 对Java程序的命名进行检查
     */
    public void checkNames(Element element) {
        nameCheckerScanner.scan(element);
    }

    class NameCheckerScanner extends ElementScanner6<Void, Void> {
        /**
         * 检查类命名是否合法
         */
        @Override
        public Void visitType(TypeElement e, Void p) {
            scan(e.getTypeParameters(), p);
            checkCamelCase(e.getSimpleName().toString(), true);
            super.visitType(e, p);
            return null;
        }

        /**
         * 检查方法命名是否合法
         */
        @Override
        public Void visitExecutable(ExecutableElement e, Void p) {

            if (e.getKind() == METHOD) {
                Name name = e.getSimpleName();
                if (name.contentEquals(e.getEnclosingElement().getSimpleName())) {
                    messager.printMessage(Diagnostic.Kind.WARNING, "方法 '" + name + "' 不应该与类名重复", e);
                    checkCamelCase(e.getSimpleName().toString(), false);
                }
            }
            super.visitExecutable(e, p);
            return null;
        }

        /**
         * 检查变量命名是否合法
         */
        @Override
        public Void visitVariable(VariableElement e, Void aVoid) {
            // 如果是枚举或常量，则必须大写，否则小驼峰
            if (e.getKind() == ENUM_CONSTANT || e.getConstantValue() != null || checkConstant(e)) {
                checkAllCaps(e.getSimpleName().toString());
            } else {
                checkCamelCase(e.getSimpleName().toString(), false);
            }
            return null;
        }


        /**
         * 判断一个变量是否常量
         */
        private boolean checkConstant(VariableElement e) {
            if (e.getEnclosingElement().getKind() == INTERFACE) {
                return true;
            } else if (e.getKind() == FIELD && e.getModifiers().containsAll(EnumSet.of(PUBLIC, STATIC, FINAL))) {
                return true;
            } else {
                return false;
            }
        }

        /**
         * 检查是否符合驼峰
         *
         * @param b 是否大驼峰
         */
        private void checkCamelCase(String name, boolean b) {
            System.out.println("检查是否符合驼峰: " + name);
        }

        /**
         * 检查是否全部大写
         */
        private void checkAllCaps(String name) {
            System.out.println("检查是否全部大写: " + name);
        }
    }
}
```


### 4.3 BADLY_NAMED_CODE： 被检查的代码


> 一段自定义的命名不规范的代码 `BADLY_NAMED_CODE.java`


```java
/**
 * 包含多出不规范命名的代码样例
 * 类名未遵循大驼峰
 */
public class BADLY_NAMED_CODE {

    /**
     * 枚举大驼峰
     */
    enum colors {
        // 枚举值要大写
        red,
        blue,
        green
    }

    /**
     * 常量大写
     */
    static final int _FORTY_XXX = 43;
    /**
     * 变量未遵循小驼峰
     */
    public static int NOt_A_CONSTANT = _FORTY_XXX;

    /**
     * 方法名与类名相同
     */
    protected void BADLY_NAMED_CODE(){
        return;
    }

    /**
     * 方法没有遵循小驼峰
     */
    public void NOTcamelCASEEmxxx() {
        return;
    }
}
```


### 4.4 使用插入式注解处理器编译文件


> 上述三个java文件在同一目录，在该目录下打开终端，输入以下命令:


```bash
# 设置当前目录为类路径，否则无法编译
export CLASSPATH=.:$CLASSPATH
# 编译NameChecker.java
javac NameChecker.java
# 编译NameCheckProcessor.java
javac NameCheckProcessor.java 
# 通过插件编译BADLY_NAMED_CODE.java
javac -processor NameCheckProcessor BADLY_NAMED_CODE.java
```


> 接下终端会输出以下信息:

```bash
检查是否符合驼峰: BADLY_NAMED_CODE
检查是否符合驼峰: colors
检查是否符合驼峰: name
检查是否全部大写: red
检查是否全部大写: blue
检查是否全部大写: green
检查是否符合驼峰: args
检查是否全部大写: _FORTY_XXX
检查是否符合驼峰: NOt_A_CONSTANT
检查是否符合驼峰: BADLY_NAMED_CODE
警告: 来自注释处理程序 'NameCheckProcessor' 的受支持 source 版本 'RELEASE_6' 低于 -source '1.8'
BADLY_NAMED_CODE.java:32: 警告: 方法 'BADLY_NAMED_CODE' 不应该与类名重复
    protected void BADLY_NAMED_CODE(){
                   ^
2 个警告
```