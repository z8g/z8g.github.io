---
layout: mypost
title: ASM 运行时增强技术
categories: [代码性能]
---

> **程序分析**、**程序生成**和**程序转换**都是非常有用的技术，可在许多应用环境下使用；ASM 用于**运行时**类生成与转换(处理**经过编译**的 Java 类)。

[TOC]


## 1 概述
- **使用范围**：严格限制于类的读、写、转换和分析 (类的加载过程就超出了它的范围之外)
- **API 模型**：ASM 提供了两个用于生成和转换已编译类的 API，一个是 core API，以基于**事件**的形式来表示类，另一个是 tree API，以基于**对象**的形式来表示类
	-  core API 要快于 tree API，因为不需要在内存中创建和存储`Object Tree`
	- 但是使用 core API 时，类转换的实现要更难一些，因为在任意给定时刻，类中只有一个元素可用，tree API 则可以在内存中获取整个类
- **体系结构**：
	- 基于事件的 API 围绕事件生成器(类分析器)、事件使用器(类写入器)和各种预定义的事件筛选器进行，可由开发者自定义，分为两个步骤：
		- 将事件生成器(ClassReader)、筛选器(ClassVisitor)和使用器(ClassWriter)组装为体系结构
		- 启动事件生成器(ClassReader)，以执行生成或转换过程
	- 基于对象的 API 中，用于操作 Object Tree 的类生成器或转换器组件是可以通过组装而形成的，它们之间的链接代表着转换的顺序
- **组织形式**：
	- `org.objectweb.asm` 和 `org.objectweb.asm.signature` 定义了基于事件的 API，并提供了类分析器和写入器组件 (asm.jar)
	- `org.objectweb.asm.util` 提供了各种基于 core API 的工具，可以在开发和调试 ASM 应用程序时使用 (asm-util.jar)
	- `org.objectweb.asm.commons` 提供了几个很有用的预定义类转换器，大多基于 core API (asm-commons.jar)
	- `org.objectweb.asm.tree` 定义了基于对象的 API，并提供了一些工具，用于在基于事件和基于对象的表示方法之间进行转换 (asm-tree.jar)
	- `org.objectweb.asm.tree.analysis` 以 tree API 为基础，提供了一个类分析框架和几个预定义的类分析器 (asm-analysis.jar)


## 2 已编译类的结构表示


一个**已编译类**和**源文件类**有以下几点区别：
- 已编译类仅描述**一个类**，一个源文件中可以包含几个类
- 已编译类**不包含注释(comment)**，但可以包含类、字段方法和代码属性(Attribute)，Java 5 引入注解(annotaion)后，属性已经变得没有什么用处了
- 已编译类中不包含 **package** 和 **import**，因此所有类型(Type)都是全限定的(Qualified)
- 已编译类中包含 [常量池(constant pool)](#)，其中包含了在类中出现的所有数值、字符串、和类型常量 (但 ASM 隐藏了与常量池有关的所有细节)

### 2.1 内部名 (InternalName)
类型(Type)只能是类(class)或接口类型(interface)，类型在已编译类中用**内部名(InternalName)** 表示；一个类的内部名就是这个类的完全限定名，`.` 号用 `/` 表示，例如 **String** 的内部名为 `java/lang/String`


### 2.2 类型描述符 (Type Descriptor)
内部名只能用于表示类或接口类型，所有其他类型，在已编译类中都是用类型描述符(Type Descriptor)表示，如 `表2.2.1` 所示。


<blockquote id="表2.2.1">表2.2.1 - 类型描述符示例</blockquote>


Java 类型 | 类型描述符 | 补充说明
---- | ----
boolean | Z |
char | C | 
byte | B | 
short | S | 
int | I | 
float | F | 
long | J | 
double | D | 
Object  | Ljava/lang/Object; | 用 `L**;` 表示一个类，以 `;` 号结尾
int[] | [I | 数组类型的描述符是一个 `[` 号后加上其**组件类型**的描述符
Object[][] | [[Ljava/lang/Obejct; |


### 2.3 方法描述符(Method Descriptor)
方法描述符是一个类型描述符列表，用一个字符串描述一个方法的参数类型(parameter Type)和返回类型(return Type)，如 `表2` 所示。


<blockquote id="表2.2.2">表2.2.2 - 方法描述符示例</blockquote>


源文件中的方法声明 | 方法描述符
---- | ----
void m(int i, float f) | (IF)V
int m(Object o) | (Ljava/lang/Object;)I
int[] m (int i, String s) | (ILjava/lang/String;)[I
Object m(int[] arr) | ([I)Ljava/lang/Object;


## 3 与已编译类相关的 ASM API


ASM 提供了三个基于 ClassVisitor API 的核心组件，用于生成和转换类：
- **ClassReader** 可以看作一个事件产生器，可以分析 `byte[]` 形式的已编译类，并调用 accept 方法入参里 ClassVisitor 实例的 visitXxx 方法 (`void ClassReader#accept(ClassVisitor, Attribute[], int)`)
- **ClassWriter** 可以看作一个事件使用器，其直接以二进制形式生成已编译类 (`byte[] ClassWriter#toByteArray()`)
- **ClassVistor** 可以看作一个事件筛选器，可以将它收到的所有**方法调用**都委托给另一个 ClassVisitor 类


### 3.1 ClassVisitor：类的生成与转换


用于生成和转换已编译类的 ASM API 是基于 **ClassVisitor** 抽象类的：

<blockquote id="清单3.1.1">清单3.1.1 - ClassVisitor 源码</blockquote>


```java
public abstract class ClassVisitor {
    protected final int api;
    protected ClassVisitor cv;
    public ClassVisitor(final int api) {...}
    public ClassVisitor(final int api, final ClassVisitor cv) {...}
    public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {...}
    public void visitSource(String source, String debug) {...}
    public void visitOuterClass(String owner, String name, String desc) {...}
    public AnnotationVisitor visitAnnotation(String desc, boolean visible) {...}
    public AnnotationVisitor visitTypeAnnotation(int typeRef, TypePath typePath, String desc, boolean visible) {...}
    public void visitAttribute(Attribute attr) {...}
    public void visitInnerClass(String name, String outerName, String innerName, int access) {...}
    public FieldVisitor visitField(int access, String name, String desc, String signature, Object value) {...}
    public MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions) {...}
    public void visitEnd() {...}
}
```



**ClassVisitor 类的方法必须按以下顺序调用：**
1. 首先调用 visit
2. 然后是对 visitSource 的最多一次调用
3. 接下来是对 visitOuterClass 的最多一次调用
4. 然后可按任意顺序对 visitAnnotation 和 visitAttribute 的任意多次调用
5. 接着可按任意顺序对 visitInnerClass、visitField 或 visitMethod 的任意多次调用
6. 最后是对 visitEnd 的一次调用

<blockquote id="清单3.1.2">清单3.1.2 - ClassVisitor 类方法的访问顺序</blockquote>


```
visit visitSource? visitOuterClass? (visitAnnotation | visitAttribute)* (visitInnerClass | visitField | visitMethod)* visitEnd
```


### 3.2 ClassReader：类的分析


在分析一个已存在的类时，唯一必需的组件是 ClassReader，以下是一个用例。

<blockquote id="清单3.2.1">清单3.2.1 - 一个用于读取 HashMap 类信息的 ClassReader 示例</blockquote>


```java
public class PrinterClassVisitor extends ClassVisitor {
    public static void main(String[] args) throws IOException {
        PrinterClassVisitor cv = new PrinterClassVisitor();
        ClassReader cr = new ClassReader("java.util.HashMap");
        cr.accept(cv, 0);
    }

    public PrinterClassVisitor() {
        super(Opcodes.ASM4);
    }

    @Override
    public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
        System.out.println(name + " extends " + superName + " {");
    }

    @Override
    public FieldVisitor visitField(int access, String name, String desc, String signature, Object value) {
        System.out.println(" " + desc + " " + name);
        return null;
    }

    @Override
    public MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions) {
        System.out.println(" " + name + desc);
        return null;
    }

    @Override
    public void visitEnd() {
        System.out.println("}");
    }
}

/* 控制台输出：
java/util/HashMap extends java/util/AbstractMap {
 J serialVersionUID
 I DEFAULT_INITIAL_CAPACITY
 ...
 hash(Ljava/lang/Object;)I
 ...
 internalWriteEntries(Ljava/io/ObjectOutputStream;)V
}
 */
```


### 3.3 ClassWriter：类的生成


为了生成一个类，唯一必需的组件是 ClassWriter，以下是一个用 ClassWriter 生成 A 接口的例子。

> 清单3.3.1 - `pkg.A` 接口


```java
package pkg;
public interface A extends Runnable {
	int field1 = -1;
	int field2 = 0;
	int method1 = (Object o);
}
```


> 清单3.3.1 - 用于生成接口 `pkg/A` 的 ClassWriter 用例


```
public class ClassWriterTest {
    public static void main(String[] args) {
        int jdkVersion = V1_5;
        int accessFlags = ACC_PUBLIC + ACC_ABSTRACT + ACC_INTERFACE;
        String name = "pkg/A";
        String signature = null;
        String superName = "java/lang/Object";
        String[] interfaces = new String[]{"java/lang/Runnable"};

        ClassWriter cw = new ClassWriter(0);
        cw.visit(jdkVersion, accessFlags, name, signature, superName, interfaces);
        cw.visitField(ACC_PUBLIC + ACC_STATIC + ACC_FINAL, "field1", "I", null, -1).visitEnd();
        cw.visitField(ACC_PUBLIC + ACC_STATIC + ACC_FINAL, "field2", "I", null, 0).visitEnd();
        cw.visitMethod(ACC_PUBLIC + ACC_ABSTRACT, "method1", "(Ljava/lang/Object;)I", null, null).visitEnd();
        cw.visitEnd();

        // 使用生成的类
        // cw.toByteArray();
    }
}
```


**使用 ClassWriter 生成的类：**


`cw.toByteArray()` 返回的字节数组可以存储到 `A.class` 文件中，也可以通过**类加载器(ClassLoader)**动态加载之。


> 清单3.3.2 - 通过自定义类加载器使用 `ClassWriter` 生成的类


```java

public class ClassWriterTest {

    public static void main(String[] args) {
        ClassWriter cw = new ClassWriter(0);
		// 生成类 ...

        // 使用生成的类
        byte[] bytes = cw.toByteArray();
        AClassLoader cl = new AClassLoader();
        Class classA = cl.defineClass("pkg.A", bytes);

        // 使用 classA ...
    }

    private static class AClassLoader extends ClassLoader {
        public Class defineClass(String name, byte[] bytes) {
            return defineClass(name, bytes, 0, bytes.length);
        }
    }
}
```

### 3.4 转换类 (transfor)

(1) 将 ClassReader 产生的事件转给 ClassWriter

<blockquote id="清单3.4.1">清单3.4.1</blockquote>


<pre style="font-family: 'Courier New','MONACO'">
byte[] in = readClassByte();
ClassReader cr = new ClassReader(in);
ClassWriter cw = new ClassWriter(0);
cr.accept(cw, 0);
byte[] out = cw.toByteArray();
</pre>


(2) 在 ClassReader 和 ClassWriter 之间引入一个 ClassVisitor

<blockquote id="清单3.4.2">清单3.4.2</blockquote>


<pre style="font-family: 'Courier New','MONACO'">
byte[] in = readClassByte();
ClassReader cr = new ClassReader(in);
ClassWriter cw = new ClassWriter(0);
<b>ClassVisitor cv = newClassVistor(cw);
cr.accept(cv, 0);</b>
byte[] out = cw.toByteArray();
</pre>


(3) 重写 ClassVisitor 的方法，实现相应的功能
<blockquote id="清单3.4.3">清单3.4.3</blockquote>


<pre style="font-family: 'Courier New','MONACO'">
private static ClassVisitor newClassVistor(ClassWriter cw) {
    return new AClassVisitor(cw);
}
private static class AClassVisitor extends ClassVisitor {
    public AClassVisitor(ClassVisitor cw) {
        super(ASM4, cw);
    }

    @Override
    public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
        super.visit(<b>V1_5</b>, access, name, signature, superName, interfaces); // 可以修改版本号
    }
}
</pre>


(4) 上面的代码中,整个 `in` 都被**分析**，并**重新构建**了 `out`，如果**将 in 中不被转换的部分直接拷贝到 out 中，不对其分析，也不生成相应的事件**，效率会高得多；ASM 会自动为方法执行这一优化：
- 在 ClassReader#accept 中传入了 ClassVisitor，如果返回的 MethodVisitor 来自一个 ClassWriter，则整个方法的内容将不会被转换
- 在这种情况下，ClassReader 不会分析这个方法的内容，也不会生成相应的事件，只是复制 ClassWriter 中表示这个方法的字节数组

<blockquote id="清单3.4.4">清单3.4.4</blockquote>



<pre style="font-family: 'Courier New','MONACO'">
byte[] in = readClassByte();
ClassReader cr = new ClassReader(in);
<b>ClassWriter cw = new ClassWriter(cr, 0);</b> //执行这一优化
ClassVisitor cv = newClassVistor(cw);
cr.accept(cv, 0);
byte[] out = cw.toByteArray();
</pre>


### 3.5 移除类成员

在重写 ClassVisitor 的方法时，**不转发相应的调用**，可以**移除**相应的类成员(member)。


<blockquote id="清单3.5.1">清单3.5.1 - 一个移除类字段的例子</blockquote>


<pre style="font-family: 'Courier New','MONACO'">
class RemoveMethodAdapter extends ClassVisitor {
    <b>private String mName;
    private String mDesc;</b>

    public RemoveMethodAdapter(ClassVisitor cv, String mName, String mDesc) {
        super(ASM4, cv);
        this.mName = mName;
        this.mDesc = mDesc;
    }

    @Override
    public MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions) {
        <b>if (name.equals(mName) && desc.equals(mDesc)) {
            return null; // 不要委托至下一个访问器，这样将移除该方法
        }</b>
        return cv.visitMethod(access, name, desc, signature, exceptions);
    }
}
</pre>




### 3.6 增加类成员
在重写 ClassVisitor 的方法时，**遵循 `2.1` 的规则，多转发一些调用**，可以**增加**相应的类成员(member)。


例如，如果要向类中添加一个字段，必须在原方法调用之间添加对 visitField 的一个新调用，而且必须将这个新调用放在类适配器的一个访问方法中(可以在 **visitEnd** 中添加字段，确保字段名称不会重复)，以下是一个在类中添加字段的用例：


<blockquote id="清单3.6.1">清单3.6.1 - 一个增加类字段的例子</blockquote>



<pre style="font-family: 'Courier New','MONACO'">
class AddFieldAdapter extends ClassVisitor {
    private int fAcc;
    private String fName;
    private String fDesc;
    private boolean isFieldPresent;

    public AddFieldAdapter(ClassVisitor cv, int fAcc, String fName, String fDesc) {
        super(ASM4, cv);
        this.fAcc = fAcc;
        this.fName = fName;
        this.fDesc = fDesc;
    }

    @Override
    public FieldVisitor visitField(int access, String name, String desc, String signature, Object value) {
        <b>if (name.equals(fName)) {
            isFieldPresent = true; // 确保字段不会重复
        }</b>
        return cv.visitField(access, name, desc, signature, value);
    }

    @Override
    public void visitEnd() {
        <b>if (!isFieldPresent) {
            FieldVisitor fv = cv.visitField(fAcc, fName, fDesc, null, null);
            if (fv != null) { // 一个类访问器可以在 visitEnd 中返回 null
                fv.visitEnd();
            }
        }</b>
        cv.visitEnd();
    }
}
</pre>




### 3.7 转换链


将几个适配器链接在一起，可以组成几个独立的类转换，以完成复杂转换。




<blockquote id="清单3.7.1">清单3.7.1 - 通过编写一个 ClassVisitor 将接收到的方法调用同时转发给几个 ClassVisitor</blockquote>



<pre style="font-family: 'Courier New','MONACO'">
class MultiClassAdapter extends ClassVisitor {
    <b>protected ClassVisitor[] cvs;</b>

    public MultiClassAdapter(ClassVisitor[] cvs) {
        super(ASM4);
        this.cvs = cvs;
    }

    @Override
    public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
        <b>for (ClassVisitor cv : cvs) {
            cv.visit(version, access, name, signature, superName, interfaces);
        }</b>
    }
}
</pre>


> 清单3.7.2 - 一个相对复杂的转换链示意图


<pre style="font-family: 'Courier New','MONACO'">
CR → [ ]       [ ] → [ ]
         ↘   ↗          ↘
          [ ] →   [ ]  →  [ ]
CR → [ ] ↗          ↘        ↘
                       ↘        CW
                         ↘    ↗
CR →              [ ]  →  [ ]
</pre>




## 4 已编译方法的结构表示


> 在已编译类的内部，方法的代码存储为一系列**字节码**指令，为了生成和转换类，最根本的办法就是**要了解这些指令，并理解它们是如何工作的**




### 4.1 执行模式


- Java 代码在**线程**内部执行
- 每个线程都有自己的执行栈，栈由**帧**组成
- 每个帧表示一个**方法调用**，每一帧包括一个**局部变量**部分和一个**操作数栈**部分
- 局部变量部分与操作数栈部分的大小取决于方法的代码，在**编译时计算完成**，并随字节代码指令一起存储在已编译类中


以下是一个具有 3 帧的执行栈。第 1 帧包含 3 个局部变量，其操作数栈的最大值为 4，其中包含 2 个值；第 2 帧包含 2 个局部变量，操作数栈中有 2 个值；第 3 帧位于执行栈的顶端，包含 4 个局部变量和 2 个操作数。


> 清单4.1.1 - 一个具有 3 帧的执行栈


<pre style="font-family: 'Courier New','MONACO'">
——————————————————    ———————————————    ——————————————————————
|     Frame 1    |    |  Frame 2    |    |       Frame 3      |
| [L0] [L1] [L2] |    | [L0] [L1]   |    | [L0] [L1] [L2][L3] |
| [V0][V2][][]   |    | [V0][V2][]  |    | [V0][V2]           |
——————————————————    ———————————————    ——————————————————————
</pre>



- 非静态(static)方法需要保存 this 引用。对于非静态方法，在创建一个帧时，会对其初始化提供一个空栈，并用目标对象 `this` 及该方法的参数来初始化其局部变量。例如调用 `a.equals(b)`时，将创建 1 帧，前 2 个局部变量将被初始化为 a 和 b
- long 和 double 需要两个变量槽。局部变量部分和操作数栈部分的每个**槽(slot)**可以保存 **long 和 double 变量之外的任意 Java 值**。


### 4.2 字节码指令


**字节码的构成：**
字节码指令由标识该指令的**操作码**和固定数目的**参数**组成。
- 操作码是一个 unsigned byte
- 参数是静态值，确定了精确的指令行为，紧跟操作码之后




**字节码的分类：**
字节码可以分为两类，一类用于在局部变量和操作数栈之间传送值，一类仅用于操作数栈。


**用于在局部变量和操作数栈之间传送值的字节码：**
- 读取一个局部变量，并将其值压到操作树栈中，其参数是局部变量的索引 i(必须读取)：iload, lload, fload, dload, aload 
- 从操作数栈中弹出一个值，并将其值存储在由索引 i 指定的局部变量中：istore, lstore,fstore, dstore, astore


> 注：将一个值存储在局部变量中，然后再以不同类型加载之，是非法的；但是如果向局部变量中存储值，而该值不同于该局部变量中存储的当前值，却是合法的。例如 `istore  1 aload 1` 序列是非法的。


**仅用于操作数栈的字节码：**
- 用于处理 **栈**  上的值：`pop` 弹出栈顶部的值；`dup` 压入顶部栈值的一个副本；`swap` 弹出两个值，并按逆序压入之；……
- 在操作数栈压入一个 **常量** 值：`aconst_null` 压入 null；`iconst_0` 压入 int 值 0；`fconst_0` 压入 0f，`dconst_0` 压入 0d，`bipush B` 压入 byte 值 B；`sipush S` 压入 short 值 S；`ldc CST` 压入任意 int、float、long、double、String 或 class 常量 CST；……
- 从操作数栈弹出数值，进行**算术逻辑**处理后，将结果压入栈中：`*add`、`*sub`、`*mul`、`*div`、`*rem` 分别对应于 `+`、`-`、`*`、`/`、`%` 运算，其中 `*` 表示 `i`、`l`、`f` 或 `d`；还有 `<<`、`>>`、`>>>`、`|`、`&`、`^` 运算的对应指令，用于处理 int 和 long 值
- 从栈中弹出一个值，进行**类型转换**后，将结果压入栈中：`i2f`、`f2d`、`l2d` 等，将数值由一种类型转换为另一种类型；`checkcast T` 将一个引用值转换为类型 T
- 用于创建**对象**、锁定对象、检测对象类型等：例如 `new TYPE` 将一个 TYPE 类型的新对象压入栈中(TYPE 是一个内部名)
- 用于读写一个**字段**的值：`getfield OWNER NAME DESC` 弹出一个对象引用，并压入其 NAME 字段的值；`putfield OWNER NAME DESC` 弹出一个值和对象引用，并将这个值存储在它的 NAME 字段中；这两种情况中，对象必须为 OWNER 类型，字段必须为 DESC 类型；`getstatic` 和 `putstatic` 是类似指令，用于静态字段(static field)。
- 用于调用一个**方法**或构造方法(其弹出值的个数等于其方法参数个数加1(用于目标对象))，并压回方法调用的结果：`invokevirtual OWNER NAME DESC` 调用在类 OWNER 中定义的 NAME 方法，其方法描述为 DESC；`invokestatic OWNER NAME DESC` 用于调用静态方法； `invokespecial` 用于私有方法和构造方法；`invokeinterface` 用于接口中定义的方法；`invokedynamic` 用于动态方法调用机制。
- 用于读写**数组**的值：`Xaload` 弹出一个索引和一个数组，并压入此索引处该数组元素的值；`Xastore` 弹出一个值，一个索引和一个数组，并将这个值存储在数组的这一索引处；这里的 X 可以是 `i`、`l`、`f`、`d`、`a`，`b`、`c`、`s`。
- 用于无条件地或者在某一条件为真时**跳转**到到一条任意指令，用于编译 if、for、do、while、break 和 continue、switch、case 等：例如 `ifeq LABEl` 从栈中弹出一个 int 值，如果该值为 0，则跳转到 LABEL 指定的指令处(否则正常执行下条指令)；还有一些跳转指令，诸如 `ifne`、`ifge`、`tableswitch`、`lookupswitch`


### 4.3 字节码指令(执行引擎)工作原理


本节基于以下 User 类，介绍字节码指令的工作原理。


> 清单4.3.1 - 用于演示字节码指令工作原理的 <code id="清单4.3.1">User</code> 类


```java
public class User {
    private int age;

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public void checkAndSetAge(int age) {
        if (age > 0) {
            this.age = age;
        } else {
            throw new IllegalArgumentException();
        }
    }
}
```


**User#getAge 说明：**


> 清单4.3.2 - `User#getAge` 方法的字节码指令


```java
public int getAge();
    Code:
       0: aload_0
       1: getfield      #2                  // Field age:I
       4: ireturn
```


- 第 1 条指令读取局部变量0，并这个 0 值压入操作数栈
- 第 2 条指令从栈中弹出这个值，即 `this`，并将对象的 `age` 字段压入栈中，即 `this.age`
- 第 3 条指令从栈中弹出这个值，并将其返回给调用者



在这个过程中，执行帧的持续状态如下：


> 清单4.3.3 - `User#getAge` 方法的执行帧的状态


<pre style="font-family: 'Courier New','MONACO'">
(1) 初始状态     (2) aload_0 之后      (3) getfield 之后
    [this]         [this]                [this    ]
    [    ]         [this]                [this.age]
</pre>




**User#setAge 说明：**


> 清单4.3.4 - `User#setAge` 方法的字节码指令



```java
  public void setAge(int);
    Code:
       0: aload_0
       1: iload_1
       2: putfield      #2                  // Field age:I
       5: return
```


- 第 1 条指令将 `this` 压入操作数栈
- 第 2 条指令压入局部变量 1，在为这个**方法调用**创建帧期间，以 age 参数初始化该变量
- 第 3 条指令弹出这两个值，并将 int 值存储在被引用对象的 age 字段中，即存储在 `this.age` 中
- 第 4 条指令表示销毁当前执行帧(编译后强制生成)，并返回调用者



> 清单4.3.5 | `User#setAge` 方法的执行帧的状态


<pre style="font-family: 'Courier New','MONACO'">
(1) 初始状态          (2) aload_0 之后      (3) iload_1 之后      (4) putfield 之后
    [this][age]         [this][age]           [this][age]          [this][age]
    [    ][   ]         [this][   ]           [this][age]          [    ][   ]
</pre>




**User 默认的构造方法说明：**


> 清单4.3.6 - `User#<init>` 方法的字节码指令



```java
  public User();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return
```


- 第 1 条指令将 `this` 压入操作数栈中
- 第 2 条指令从栈中弹出这个值，并调用 `Object#<init>` 方法
- 第 3 条指令返回调用者




**User#checkAndSetAge 方法说明：**


> 清单4.3.7 - `User#<init>` 方法的字节码指令


```java
  public void checkAndSetAge(int);
    Code:
       0: iload_1
       1: ifle          12
       4: aload_0
       5: iload_1
       6: putfield      #2                  // Field age:I
       9: goto          20
      12: new           #3                  // class java/lang/IllegalArgumentException
      15: dup
      16: invokespecial #4                  // Method java/lang/IllegalArgumentException."<init>":()V
      19: athrow
      20: return
```

- 第 1 条指令将初始化为 age 的局部变量1 压入操作数栈
- 第 2 条指令(ifle)从栈中弹出 1 个值，并将其与 0 进行比较，如果小于等于(le) 0，则跳转到程序计数器(pc)为12的指令，否则不做任何事，继续执行下一条指令
- 第 3 ～ 5 条指令与 `setAge` 方法类似
- 第 6 条指令(goto)表示无条件跳转到 pc 为 20 的指令，即返回
- 第 7 条指令(new)表示创建一个 IllegalArgumentException 实例，并将其压入操作数栈
- 第 8 条指令(dup)表示重复这个值
- 第 9 条指令(invokespecial)表示弹出这两个副本之一，并对其调用构造方法
- 第 10 条指令(athorw)弹出剩下的副本，并将它作为异常抛出(方法异常调用完成)，之后不会继续执行下一条指令




**异常表(Exception table)：**


不存在用于捕获异常的字节码，而是将一个方法的字节码与一个**异常表(Exception table)**相关联，异常表规定了在某方法中一给定部分抛出异常时必须执行的代码。对于下面的 tryCatchFinallyTest 方法：


> 清单4.3.8 - `tryCatchFinallyTest` 方法，该方法会返回 `300`


```java
public static int tryCatchFinallyTest() {
    try {
        return 100;
    } catch (Exception ex) {
        return 200;
    } finally {
        return 300;
    }
}
```


其字节码指令如下：


> 清单4.3.9 - `tryCatchFinallyTest` 方法的字节码指令


```java
  public static int tryCatchFinallyTest();
    Code:
       0: bipush        100
       2: istore_0
       3: sipush        300
       6: ireturn
       7: astore_0
       8: sipush        200
      11: istore_1
      12: sipush        300
      15: ireturn
      16: astore_2
      17: sipush        300
      20: ireturn
    Exception table:
       from    to  target type
           0     3     7   Class java/lang/Exception
           0     3    16   any
           7    12    16   any
}
```

- `from=0, to=3, target=7, type=Exception` 表示：对于 pc 为 0 ~ 3 的指令(try 语句块)，如果发生类型为 java/lang/Exception 的异常，将跳转到 pc=7 处，否则执行 pc=3 之后的 `ireturn`
- `from=0, to=3, target=16, type=any` 表示：执行完 pc 为 0 ~ 3 的指令(try 语句块)后，无条件跳转到 pc=16 处的指令(finally 语句块)
- `from=7, to=12, target=16, type=any` 表示：执行完 pc 为 7 ~ 12 的指令(catch 语句块)后，无条件跳转到 pc=16 处的指令(finally 语句块)



## 5 与已编译方法相关的 ASM API

ASM 提供了三个基于 MethodVisitor API 的核心组件，用于生成和转换方法：
- ClassReader 可以看作一个事件产生器，可以分析已编译方法的内容 (void ClassReader#accept(ClassVisitor, Attribute[], int))
- ClassWriter 可以直接以二进制形式生成已编译方法 (MethodVisitor ClassWriter#visitMethod(access, name, desc, signature, exceptions))
- MethodVistor 可以看作一个事件筛选器，将它收到的所有方法调用都委托给另一个 MethodVistor 类


### 5.1 MethodVisitor：方法的生成与转换


用于生成和转换已编译方法的 ASM API 是基于 MethodVisitor 抽象类的：



> 清单5.1.1 - MethodVisitor 源码

```java
public abstract class MethodVisitor {
    protected final int api;
    protected MethodVisitor mv;

    public MethodVisitor(final int api) {}
    public MethodVisitor(final int api, final MethodVisitor mv) {}

    // Parameters, annotations and non standard attributes
    public void visitParameter(String name, int access) {}
    public AnnotationVisitor visitAnnotationDefault() {}
    public AnnotationVisitor visitAnnotation(String desc, boolean visible) {}
    public AnnotationVisitor visitTypeAnnotation(int typeRef, TypePath typePath, String desc, boolean visible) {}
    public AnnotationVisitor visitParameterAnnotation(int parameter, String desc, boolean visible) {}
    public void visitAttribute(Attribute attr) {}
    public void visitCode() {}
    public void visitFrame(int type, int nLocal, Object[] local, int nStack, Object[] stack) {}

    // Normal instructions
    public void visitInsn(int opcode) {}
    public void visitIntInsn(int opcode, int operand) {}
    public void visitVarInsn(int opcode, int var) {}
    public void visitTypeInsn(int opcode, String type) {}
    public void visitFieldInsn(int opcode, String owner, String name, String desc) {}
    public void visitMethodInsn(int opcode, String owner, String name, String desc) {}
    public void visitMethodInsn(int opcode, String owner, String name, String desc, boolean itf) {}
    public void visitInvokeDynamicInsn(String name, String desc, Handle bsm, Object... bsmArgs) {}
    public void visitJumpInsn(int opcode, Label label) {}
    public void visitLabel(Label label) {}

    // Special instructions
    public void visitLdcInsn(Object cst) {}
    public void visitIincInsn(int var, int increment) {}
    public void visitTableSwitchInsn(int min, int max, Label dflt, Label... labels) {}
    public void visitLookupSwitchInsn(Label dflt, int[] keys, Label[] labels) {}
    public void visitMultiANewArrayInsn(String desc, int dims) {}
    public AnnotationVisitor visitInsnAnnotation(int typeRef, TypePath typePath, String desc, boolean visible) {}

    // Exceptions table entries, debug information, max stack and max locals
    public void visitTryCatchBlock(Label start, Label end, Label handler, String type) {}
    public AnnotationVisitor visitTryCatchAnnotation(int typeRef, TypePath typePath, String desc, boolean visible) {}
    public void visitLocalVariable(String name, String desc, String signature, Label start, Label end, int index) {}
    public AnnotationVisitor visitLocalVariableAnnotation(int typeRef, TypePath typePath, Label[] start, Label[] end, int[] index, String desc, boolean visible) {}
    public void visitLineNumber(int line, Label start) {}
    public void visitMaxs(int maxStack, int maxLocals) {}

    public void visitEnd() {}
}
```


**MethodVisitor 类的方法必须按以下顺序调用：**


> 清单5.1.2 - MethodVisitor 类方法的调用顺序




```java
visitAnnotationDefault? (visitAnnotation | visitParameterAnnotation | visitAttribute)* (visitCode (visitTryCatchBlock | visitLabel | visitFrame | visitXxxInsn | visitLocalVariable | visitLineNumber)* visitMaxs)? visitEnd
```

- visitCode 和 visitMaxs 方法可用于**检测方法的字节码在一个事件序列中的开始与结束**
- visitEnd 必须在最后调用，用于检测一个方法在一个事件序列中的结束
- 可以将 ClassVisitor 和 MethodVisitor 类合并，生成完整的类


> 清单5.1.3 - 将 ClassVisitor 和 MethodVisitor 类合并，生成完整的类 [书写格式]


<pre style="font-family: 'Courier New','MONACO'">
PrinterClassVisitor cv = new PrinterClassVisitor();

MethodVisitor mv1 = cv.visitMethod(0, "m1", null, null, null);
mv1.visitCode();
mv1.visitInsn(0);
// ...
mv1.visitMaxs(0, 0);
mv1.visitEnd();

MethodVisitor mv2 = cv.visitMethod(0, "m2", null, null, null);
mv2.visitCode();
mv2.visitInsn(0);
// ...
mv2.visitMaxs(0, 0);
mv2.visitEnd();

cv.visitEnd();
</pre>



> 清单5.1.4 - 将 ClassVisitor 和 <code id="清单5.1.4">MethodVisitor</code> 类合并，生成完整的类 [代码示例]


<pre style="font-family: 'Courier New','MONACO'">
/**
 * 生成默认的构造方法
 *
 * @param superName 父类名称
 */
static void generateDefaultConstruct(ClassWriter cw, String superName) {
    MethodVisitor mv = cw.visitMethod(ACC_PUBLIC, MethodName.CONSTRUCTOR, MethodDesc.EMPTY_VOID, null, null);
    // 生成构造方法的字节码指令
    mv.visitVarInsn(ALOAD, 0);
    mv.visitMethodInsn(INVOKESPECIAL, superName, MethodName.CONSTRUCTOR, MethodDesc.EMPTY_VOID, false);
    mv.visitInsn(RETURN);
    mv.visitMaxs(1, 1);
    mv.visitEnd();
}


/**
 * 生成类方法
 */
private static void generateMethod(ClassWriter cw, Class<?> beanClass, boolean usePropertyDescriptor) {
    String internalClassName = BeanEnhanceUtils.getInternalName(beanClass.getName());
    ClassDesc classDesc = BeanEnhanceUtils.getClassDescription(beanClass, usePropertyDescriptor);
    MethodVisitor mv = cw.visitMethod(ACC_PUBLIC, MethodName.VALUE, MethodDesc.VALUE, null, null);
    mv.visitCode();

    // 有属性，需要调用 getter 方法
    if (classDesc.hasField) {
        generateMethodWithFields(internalClassName, classDesc, mv);
    } else {
        generateMethodWithNoField(mv, classDesc, internalClassName);
    }

    mv.visitEnd();
}


/**
 * 生成beanClass对应的增强类的字节流
 *
 * @param superName  父类
 * @param interfaces 接口列表
 */
static byte[] generate(Class<?> beanClass, String superName, String[] interfaces, boolean usePropertyDescriptor) throws Exception {
    String beanClassName = beanClass.getName();
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_MAXS);// 自动计算maxStack
    String getterClassName = createGeneratedClassName(beanClassName);
    cw.visit(V1_8, ACC_PUBLIC + ACC_SUPER, BeanEnhanceUtils.getInternalName(getterClassName), null, superName, interfaces);
    <b>generateDefaultConstruct(cw, superName);</b> // 生成默认构造方法
    <b>generateMethod(cw, beanClass, usePropertyDescriptor);</b> // 生成GetterClass
    cw.visitEnd();
    return cw.toByteArray();
}
</pre>




**new ClassWriter(int flag) 选项：**
- flag 为 0 时，不会进行自动计算，必须自行计算帧，局部变量与操作数栈的大小
- flag 为 `ClassWriter#COMPUTE_MAXS` 时(性能降低10%)，ASM 将自动计算局部变量与操作数栈部分的大小，还是必须调用 visitMaxs 方法(可以使用任何参数，但是会被忽略并重新计算)，必须自行计算栈帧(实例中 generateDefaultConstruct 方法便是自行计算栈帧)
- flag 为 `ClassWriter#COMPUTE_FRAMES` 时(性能降低50%)，一切都是自动计算，不再需要调用 visitFrame，但仍然必须调用 visitMaxs 方法(参数将被忽略并重新计算)


### 5.2 方法的生成

针对以下 User 类(拷贝自 [清单4.3.1](#清单4.3.1))：




> 清单5.2.1 - 用于演示 MethodVisitor 方法生成的 User 类




```java
public class User {
    private int age;

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public void checkAndSetAge(int age) {
        if (age > 0) {
            this.age = age;
        } else {
            throw new IllegalArgumentException();
        }
    }
}
```



**生成 getAge 方法的 MethodVisitor 用例写法如下：**




> 清单5.2.2 - 用于生成 `User#getAge` 方法的 MethodVisitor 用例


```java
MethodVisitor mv = getMethodVisitor();
mv.visitCode();
mv.visitVarInsn(Opcodes.ALOAD, 0);
mv.visitFieldInsn(Opcodes.GETFIELD, "pkg/User", "age", "I");
mv.visitInsn(Opcodes.IRETURN);
mv.visitMaxs(1, 1);
mv.visitEnd();
```



**生成 checkAndSetAge 方法的 MethodVisitor 用例写法如下：**


> 清单5.2.3 - 用于生成 `User#checkAndSetAge` 方法的 MethodVisitor 用例

<pre style="font-family: 'Courier New','MONACO'">
MethodVisitor mv = getMethodVisitor();                  // public void checkAndSetAge(int);
mv.visitCode();                                         // Code:

mv.visitVarInsn(Opcodes.ILOAD, 1);                      // 0: iload_1

Label thenBlockLabel = new Label(); // (pc = 12)
mv.visitJumpInsn(Opcodes.IFLE, thenBlockLabel);         // 1: ifle   12
mv.visitVarInsn(Opcodes.ALOAD, 0);                      // 4: aload_0
mv.visitVarInsn(Opcodes.ILOAD, 1);                      // 5: iload_1
mv.visitFieldInsn(Opcodes.PUTFIELD,
        "User", "age", "I");                            // 6: putfield #2 // Field age:I

Label elseBlockLabel = new Label();// (pc = 20)
mv.visitJumpInsn(Opcodes.GOTO, elseBlockLabel);         // 9: goto   20
mv.visitLabel(thenBlockLabel);
mv.visitFrame(Opcodes.F_SAME,
        0, null, 0, null);
mv.visitTypeInsn(Opcodes.NEW,
        "java/lang/IllegalArgumentException");          // 12: new #3  // class java/lang/IllegalArgumentException
mv.visitInsn(Opcodes.DUP);                              // 15: dup
mv.visitMethodInsn(Opcodes.INVOKEINTERFACE,             // 16: invokespecial #4  // Method java/lang/IllegalArgumentException."<init>":()V
        "java/lang/IllegalArgumentException",
        "<init>", "()V");                               
mv.visitInsn(Opcodes.ATHROW);                           // 19: athrow
mv.visitLabel(elseBlockLabel);
mv.visitFrame(Opcodes.F_SAME, 0, null, 0, null);
mv.visitInsn(Opcodes.RETURN);                           // 20: return
mv.visitMaxs(2, 2);

mv.visitEnd();
</pre>



## 6 通过 ASM 为模板引擎实现运行时类增强


### 6.1 模板引擎示例


假设存在以下一种使用模板引擎的场景：


1. 开发者创建了 [清单4.3.1](清单4.3.1) 中的 `User` Java 类，如 `清单6.1.1` 所示
2. 开发者在 `user.html` 文件中自定义了数据模板，如 `清单6.1.2` 所示
3. 开发者调用模板引擎，进行数据绑定与结果渲染，如 `清单6.1.3` 所示
4. 模板引擎输出渲染结果，如 `清单6.1.4` 所示



> 清单6.1.1 - 用于演示模板引擎的 User 类


```java
public class User {
    private int age;

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public void checkAndSetAge(int age) {
        if (age > 0) {
            this.age = age;
        } else {
            throw new IllegalArgumentException();
        }
    }
}
```




> 清单6.1.2 - `user.html` 文件的内容




```html
<pre>
idUserMap 中 key 为 1 所对应的 User 实例的 age = ${idUserMap[1].age}
userList 中下标为 1 所对应的 User 实例的 age = ${userList[1].age}
变量 min_age = ${min_age}
</pre>

<ul><!-- 遍历名称为  "userList" 的列表 -->
    <% for (user in userList) { %>
    <li>${user.age}</li>
    <% } %>
</ul>

<table><!-- 遍历名称为 "idUserMap" 的映射 -->
    <% for (entry in idUserMap) {%>
    <tr>
        <td>${entry.key}</td>
        <td>${entry.value.age}</td>
    </tr>
    <% } %>
</table>
```




> 清单6.1.3 - 通过模板引擎进行数据绑定并渲染结果



```java
Template t = gt.getTemplate("/user.html");

User zhangsan = new User(21);
User lisi = new User(22);
User wangwu = new User(23);

Map<Integer, User> idUserMap = new HashMap<>();
idUserMap.put(1, zhangsan);
idUserMap.put(2, lisi);

List<User> userList = Arrays.asList(zhangsan, lisi, wangwu);

t.binding("zhangsan", zhangsan);
t.binding("idUserMap", idUserMap);
t.binding("userList", userList);
t.binding("min_age", 18);
String res = t.render();

System.out.println(res);
```


> 清单6.1.4 - 模板引擎的数据渲染结果


```html
<pre>
idUserMap 中 key 为 1 所对应的 User 实例的 age = 21
userList 中下标为 1 所对应的 User 实例的 age = 22
变量 min_age = 18
</pre>

<ul><!-- 遍历名称为  "userList" 的列表 -->
    <li>21</li>
    <li>22</li>
    <li>23</li>
</ul>

<table><!-- 遍历名称为 "idUserMap" 的映射 -->
    <tr>
        <td>1</td>
        <td>21</td>
    </tr>
    <tr>
        <td>2</td>
        <td>22</td>
    </tr>
</table>
```



### 6.2 运行时生成类实例

在 `6.1` 中演示了模板引擎的使用，本文不讨论其语法处理(例如处理循环)的实现，而只是介绍模板引擎如何处理 `User` 类。主要分为以下几个步骤：
1. 将所有支持的类型(List/Map/Array/Model/...)抽象到 `AttributeAccess` 类中(参考`清单6.2.1`)，运行时生成 `User$AttributeAccess` 类的 `byte[]`，设为 `code`
2. 运行时通过自定义类加载器加载`code`(参考`清单6.2.2` 与 `清单6.2.3`)，定义 `User$AttributeAccess` 类，并返回该类的实例，设为 `user1`
3. 将 `user1` 提供给模板引擎使用


> 清单6.2.1 AttributeAccess 抽象类，封装了一个获取对象的属性的值的方法


```java
public abstract class AttributeAccess implements java.io.Serializable {
    public abstract Object value(Object o, Object name);
    public void setValue(Object o, Object name, Object value) {
        updateValue(o, name, value);
    }
	// ...
}
```


> 清单6.2.2 自定义类加载器


```java
public class ByteClassLoader extends ClassLoader {
    public ByteClassLoader(ClassLoader parent) {
        super(parent);
    }

    public Class<?> defineClass(String name, byte[] b) {
        return defineClass(name, b, 0, b.length);
    }

    public Class<?> findClassByName(String clazzName) {
        try {
            return getParent().loadClass(clazzName);
        } catch (ClassNotFoundException ignored) {
        }
        return null;
    }
}
```




> 清单6.2.3 通过自定义类加载器加载 code




```java
private Object loadContextClassLoader(byte[] code, String className) {
    Object obj;
    try {
        Class<?> enhanceClass = byteContextLoader.findClassByName(className);
        if (enhanceClass == null) {
            enhanceClass = byteContextLoader.defineClass(className, code);
        }
        obj = enhanceClass.newInstance();
    } catch (Exception ex) {
        return null;
    }
    return obj;
}
```


综上所述，问题变为: 如何通过 ASM 生成 `User$AttributeAccess` 类


### 6.3 通过 ASM 进行类的生成与转换


运行时，会通过 ASM 生成 AttributeAccess 的子类 `User$AttributeAccess`，参考 [清单5.1.4](#清单5.1.4) 中的 `generate` 方法，ASM 会为 `User$AttributeAccess` 生成以下方法：
- 无参构造方法
- value 方法，对应实现 `AttributeAccess#value(Object, Object)` 方法


> 清单6.3.1 - `generate` 方法

```java
// 生成 AttributeAccess 对应的增强类的字节数组
generate(beanClass, InternalName.ATTRIBUTE_ACCESS, null, usePropertyDescriptor)；

/**
 * 生成 beanClass 对应的增强类的字节流
 *
 * @param superName  父类
 * @param interfaces 接口列表
 */
static byte[] generate(Class<?> beanClass, String superName, String[] interfaces, boolean usePropertyDescriptor) throws Exception {
    String beanClassName = beanClass.getName();
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_MAXS);// 自动计算maxStack
    String getterClassName = createGeneratedClassName(beanClassName);
    cw.visit(V1_8, ACC_PUBLIC + ACC_SUPER, BeanEnhanceUtils.getInternalName(getterClassName), null, superName, interfaces);
    // 生成默认构造方法
    generateDefaultConstruct(cw, superName);
    // 生成类方法
    generateMethod(cw, beanClass, usePropertyDescriptor);
    cw.visitEnd();
    return cw.toByteArray();
}
```



并为其实现 `AttributeAccess#value(Object, Object)` 方法，分三种情况：
- User 中没有 get(String) 或 get(Object) 方法，如 `清单6.3.2` 所示
- User 中包含 get(String) 方法，如 `清单6.3.3` 所示
- User 中包含 get(Object) 方法，如 `清单6.3.4` 所示




> 清单6.3.2 - User 中没有 get(String) 或 get(Object) 方法时，ASM 生成的 value 方法 [示例代码]



<pre style="font-family: 'Courier New','MONACO'">
public Object value(Object bean, Object attr) {
	String attrStr = attr.toString();
    int hash = attrStr.hashCode();
    User user = (User) bean;
    switch (hash) {
    	case 1:
    		return user.getAge();
        case 2:
        	return user.getXxx();
        case 3:
            if("age".equals(attrStr)){
                return user.getAge();
            }
            if("xxx".equals(attrStr)){
      			return user.getXxx();
            }
   }
   <b>throw new RuntimeException(ATTRIBUTE_NOT_FOUND, "attribute : " + attrStr);</b>
}
</pre>




> 清单6.3.3 - User 中包含 get(String) 方法时，ASM 生成的 value 方法 [示例代码]


<pre style="font-family: 'Courier New','MONACO'">
public Object value(Object bean, Object attr) {
	String attrStr = attr.toString();
    int hash = attrStr.hashCode();
    User user = (User) bean;
    switch (hash) {
    	case 1:
    		return user.getAge();
        case 2:
        	return user.getXxx();
        case 3:
            if("age".equals(attrStr)){
                return user.getAge();
            }
            if("xxx".equals(attrStr)){
      			return user.getXxx();
            }
    }
    <b>return user.get(attrStr);</b>
}
</pre>




> 清单6.3.4 - User 中包含 get(Object) 方法时，ASM 生成的 value 方法 [示例代码]


<pre style="font-family: 'Courier New','MONACO'">
public Object value(Object bean, Object attr) {
	String attrStr = attr.toString();
    int hash = attrStr.hashCode();
    User user = (User) bean;
    switch (hash) {
    	case 1:
    		return user.getAge();
        case 2:
        	return user.getXxx();
        case 3:
            if("age".equals(attrStr)){
                return user.getAge();
            }
            if("xxx".equals(attrStr)){
      			return user.getXxx();
            }
    }
    <b>return user.get(attr);</b>
}
</pre>


至此，`User$AttrubiteAccess` 类的生成与转换的过程结束，返回的 `byte[]` 将传给自定义的类加载器。


## 附录
### 附录A 元数据的结构表示

> 表A-1 - 类型签名举例


Java 类型 | 对应的类型签名
---- | ----
`List<E>` | `Ljava/util/List<TE;>;`
`List<?>` | `Ljava/util/List<*>;`
`List<? extends Number>` | `Ljava/util/List<+Ljava/lang/Number;>;`
`List<? super Integer>` | `Ljava/util/List<-Ljava/lang/Integer;>;`
`List<List<String>[]>` | `Ljava/util/List<[Ljava/util/List<Ljava/lang/String;>;>;`
`HashMap<K, V>.HashIterator<K>` | `Ljava/util/HashMap<TK;TV;>.HashIterator<TK;>;`
`static <T> Class<? extends T> m (int n)` | `<T:Ljava/lang/Obejct;>(I)Ljava/lang/Class<+TT>;`



> 表A-2 - 类签名举例


Java 类 | 对应的类签名
---- | ----
`C<E> extends List<E>` | `<E:Ljava/lang/Object;>Ljava/util/List<TE;>;`


