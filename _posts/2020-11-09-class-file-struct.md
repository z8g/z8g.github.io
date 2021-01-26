---
layout: mypost
title: class 文件结构
categories: [Java基础]
---

> 2020年10月份，《Java虚拟机规范(Java SE 8版)》读书笔记

[TOC]

## 1 概述


本文主要介绍 [javac](#) 编译 `*.java` 文件后，生成的 `*.class` 文件的结构。


### 1.1 unsigned byte 解析时的转换


class 文件中使用 unsigned byte，也就是说解析时需要将 byte 转换成 short 表示，转换方法是：


```java
byte rawValue = -2;
short value = (short) ((short) rawValue & 0xFF); // 254
```


例如在 class 文件中的 2，解析时会表示成 2，class 文件中的 -2，解析时便表示成 254；




### 1.2 ClassFile 结构


每一个 class 文件对应一个如下的 ClassFile 结构，如下所示。
其中 u2 表示 2 个 unsigned byte，u4 表示 4 个 unsigned byte，以此类推；
每行注释中，第一个中括号里面的内容表示数组的长度，第二个中括号里面表示该字段的名称，剩下的内容是该字段的含义。

```
/** [u4] [magic] 魔数(必须为十六进制CABEBABE) */
byte[] magic;
/** [u2] [minor_version] 次版本号 */
byte[] minorVersion;
/** [u2] [major_version] 主版本号 */
byte[] majorVersion;

/** [u2] [constant_pool_count] 常量池的长度 */
byte[] constPoolCount;
/** [constant_pool_count-1] [constant_pool] 常量池 */
ConstInfo[] constPool;

/** [u2] [access_flags] 访问标记 */
byte[] accessFlags;
/** [u2] [this_class] this 类的常量池索引 */
byte[] thisClass;
/** [u2] [super_class] super 类的常量池索引 */
byte[] superClass;

/** [u2] [interfaces_count] 接口列表的长度 */
byte[] interfacesCount;
/** [interfaces_count] [interfaces] 接口列表 */
InterfaceTable[] interfaces;

/** [u2] [fields_count] 字段列表的长度 */
byte[] fieldsCount;
/** [fields_count] [fields] 字段列表 */
FieldTable[] fields;

/** [u2] [methods_count] 方法列表的长度 */
byte[] methodsCount;
/** [methods_count] [methods] 方法列表 */
MethodTable[] methods;

/** [u2] [attributes_count] 属性列表的长度 */
byte[] attributesCount;
/** [attributes_count] [attributes] 属性列表 */
Attr[] attributes;
```


## 2 ClassFile 解析程序


笔者根据规范以及《深入理解Java虚拟机(第3版)》，编写了 ByteCode.java 程序，实现了 class 文件的解析。


### 2.1 程序的初始化


magic、minorVersion 等长度都是固定的，在 ByteCode 初始化时便对其进行实例化。content 持有字节流的引用，pos 则表示当前读取到的位置。




```java
/** class 无符号字节数组的引用 */
private final byte[] content;
/** 读取 {@link #content} 时的当前位置 */
private int pos = 0;

/**
 * 构造方法
 *
 * @param unsignedBytes 通过任意形式传入的无符号字节数组
 */
public ByteCode(byte[] unsignedBytes) {
    magic = new byte[U4];
    minorVersion = new byte[U2];
    majorVersion = new byte[U2];
    constPoolCount = new byte[U2];
    accessFlags = new byte[U2];
    thisClass = new byte[U2];
    superClass = new byte[U2];
    interfacesCount = new byte[U2];
    fieldsCount = new byte[U2];
    methodsCount = new byte[U2];
    attributesCount = new byte[U2];

    this.content = unsignedBytes;
}
```


### 2.2 各结构的解析过程总览


在进行初始化后，根据规范，按照顺序解析字节码，并对 pos 同步更改。


```java
/**
 * 严格地按照顺序解析字节码
 */
public void parseByteCode() {
    pos += read(content, pos, magic);               // 读取 魔数
    pos += read(content, pos, minorVersion);        // 读取 次版本号
    pos += read(content, pos, majorVersion);        // 读取 主版本号

    pos += read(content, pos, constPoolCount);      // 读取 常量池的长度
    readConstPoolInfo();                            // 读取 常量池

    pos += read(content, pos, accessFlags);         // 读取 访问标记
    pos += read(content, pos, thisClass);           // 读取 this
    pos += read(content, pos, superClass);          // 读取 super

    pos += read(content, pos, interfacesCount);     // 读取 接口列表的长度
    readInterfaces();                               // 读取 接口列表

    pos += read(content, pos, fieldsCount);         // 读取 字段列表的长度
    readFields();                                   // 读取 字段列表

    pos += read(content, pos, methodsCount);        // 读取 方法列表的长度
    readMethods();                                  // 读取 方法列表

    pos += read(content, pos, attributesCount);     // 读取 属性列表的长度
    readAttributes();                               // 读取 属性列表

    if (pos != content.length) {                    // 读取完毕后校验是否按照JVM规范读取完毕
        throw new IllegalStateException("解析的数据与规范不符！");
    }
}
```


read 方法的作用是拷贝读取的内容到 target 数组，并返回读取的长度，以对 pos 更新


```java
/**
 * 滚动地读取字节，从 {@param source} 滚动读取字节，填充到 {@param target} 中
 *
 * @param source    数据源
 * @param sourcePos 数据源的开始下标
 * @param target    目标字节数组
 * @return 返回读取的字节数
 */
public static int read(byte[] source, int sourcePos, byte[] target) {
    int targetLength = target.length;
    System.arraycopy(source, sourcePos, target, 0, targetLength);
    return targetLength;
}
```


接下来，还需读取以下内容：
- readConstPoolInfo
- readFields
- readMethods
- readAttributes


## 3 读取常量池(constant_pool)

### 3.1 tag 位

常量池表中的所有的项具有如下的通用格式：




```java
cp_info {
    u1 tag;
	u1 info[];
}
```


其中 tag 对应规则如下：


```
int TAG_UTF8 = 1;
int TAG_INTEGER = 3;
int TAG_FLOAT = 4;
int TAG_LONG = 5;
int TAG_DOUBLE = 6;
int TAG_CLASS = 7;
int TAG_STRING = 8;
int TAG_FIELDREF = 9;
int TAG_METHODREF = 10;
int TAG_INTERFACE_METHODREF = 11;
int TAG_NAME_AND_TYPE = 12;
int TAG_METHOD_HANDLE = 15;
int TAG_METHOD_TYPE = 16;
int TAG_DYNAMIC = 17;
int TAG_INVOKE_DYNAMIC = 18;
int TAG_MODULE = 19;
int TAG_PACKAGE = 20;
```

### 3.2 常量池解析过程总览

常量池的解析过程，总结下来，分为以下几步：
- 获取常量池项的长度
- 根据其长度，遍历去识别每个 tag ，再对每个类型的 tag 去作对应的解析


以下分别是获取常量池长度的实现，和遍历读取常量池项的实现。


```java
/**
 * 返回常量池长度的数值表示
 *
 * @param constPoolCount 表示常量池长度的字节数组
 * @return 常量池长度的数值表示。常量池不同于Java语言习惯，是从1开始计数的。假设常量池长度为2，则只包含1,2两个常量项
 */
private static int valueOfConstPoolLength(byte[] constPoolCount) {
    return unsignedBytes2Int(constPoolCount) - 1;
}

/**
 * 读取常量池
 */
private void readConstPoolInfo() {
    constPool = new ConstInfo[valueOfConstPoolLength(constPoolCount)];

    for (int i = 0; i < constPool.length; i++) {
        byte[] tag = new byte[U1]; // tag u1
        pos += Util.read(content, pos, tag);

        int tagValue = unsignedBytes2Int(tag);
        switch (tagValue) {
            case IConstPoolInfo.TAG_UTF8:
                constPool[i] = new Utf8Const(content, pos, tagValue);
                break;
            case IConstPoolInfo.TAG_INTEGER:
                constPool[i] = new IntegerConst(content, pos, tagValue);
                break;
            case IConstPoolInfo.TAG_FLOAT:
                constPool[i] = new FloatConst(content, pos, tagValue);
                break;
            case IConstPoolInfo.TAG_LONG:
                constPool[i] = new LongConst(content, pos, tagValue);
                break;
            case IConstPoolInfo.TAG_DOUBLE:
                constPool[i] = new DoubleConst(content, pos, tagValue);
                break;
            case IConstPoolInfo.TAG_CLASS:
                constPool[i] = new ClassConst(content, pos, tagValue);
                break;
            case IConstPoolInfo.TAG_STRING:
                constPool[i] = new StringConst(content, pos, tagValue);
                break;
            case IConstPoolInfo.TAG_FIELDREF:
                constPool[i] = new FieldrefConst(content, pos, tagValue);
                break;
            case IConstPoolInfo.TAG_METHODREF:
                constPool[i] = new MethodrefConst(content, pos, tagValue);
                break;
            case IConstPoolInfo.TAG_INTERFACE_METHODREF:
                constPool[i] = new InterfaceMethodrefConst(content, pos, tagValue);
                break;
            case IConstPoolInfo.TAG_NAME_AND_TYPE:
                constPool[i] = new NameAndTypeConst(content, pos, tagValue);
                break;
            case IConstPoolInfo.TAG_METHOD_HANDLE:
                constPool[i] = new MethodHandleConst(content, pos, tagValue);
                break;
            case IConstPoolInfo.TAG_METHOD_TYPE:
                constPool[i] = new MethodTypeConst(content, pos, tagValue);
                break;
            case IConstPoolInfo.TAG_DYNAMIC:
                constPool[i] = new DynamicConst(content, pos, tagValue);
                break;
            case IConstPoolInfo.TAG_INVOKE_DYNAMIC:
                constPool[i] = new InvokeDynamicConst(content, pos, tagValue);
                break;
            case IConstPoolInfo.TAG_MODULE:
                constPool[i] = new ModuleConst(content, pos, tagValue);
                break;
            case IConstPoolInfo.TAG_PACKAGE:
                constPool[i] = new PackageConst(content, pos, tagValue);
                break;
            default:
                throw new IllegalStateException("无效tag：" + tagValue);
        }
        pos += constPool[i].getOffset();
    }
}
```

以下是常量池各项的结构：


```txt
CONSTANT_Utf8_info {
	u1 tag;								// 标志位，必须为 1
	u2 utf8_string_length;				// Utf8 编码的字符串的长度
	utf8_string_length utf8_string;		// Utf8 编码的字符串
}

CONSTANT_Integer_info {
	u1 tag;								// 标志位，必须为 3
	u4 integer_value;					// Integer 类型的数值表示
}

CONSTANT_Float_info {
	u1 tag;								// 标志位，必须为 4
	u4 float_value;						// Float 类型的数值表示
}

CONSTANT_Long_info {
	u1 tag;								// 标志位，必须为 5
	u8 long_value;						// Long 类型的数值表示
}

CONSTANT_Double_info {
	u1 tag;								// 标志位，必须为 6
	u8 double_value;					// Double 类型的数值表示
}

CONSTANT_Class_info {
	u1 tag;								// 标志位，必须为 7
	u2 class_name_index;				// 指向全限定名常量项的索引
}

CONSTANT_String_info {
	u1 tag;								// 标志位，必须为 8
	u2 string_literal_index;			// 指向字符串字面量的常量池索引
}

CONSTANT_Fieldref_info {
	u1 tag;								// 标志位，必须为 9
	u2 class_info_index;				// 指向声明方法的类描述符 CONSTANT_Class_info 的常量池索引
	u2 name_and_type_info_index;		// 指向字段描述符 CONSTANT_NameAndType_info 的常量池索引
}

CONSTANT_Methodref_info {
	u1 tag;								// 标志位，必须为 10
	u2 class_info_index;				// 指向声明方法的类描述符 CONSTANT_Class_info 的常量池索引
	u2 name_and_type_info_index;		// 指向字段描述符 CONSTANT_NameAndType_info 的常量池索引
}

CONSTANT_InterfaceMethodref_info {
	u1 tag;								// 标志位，必须为 11
	u2 class_info_index;				// 指向声明方法的类描述符 CONSTANT_Class_info 的常量池索引
	u2 name_and_type_info_index;		// 指向字段描述符 CONSTANT_NameAndType_info 的常量池索引
}

CONSTANT_NameAndType_info {
	u1 tag;								// 标志位，必须为 12
	u2 name_index;						// 指向该字段或方法 名称常量项 的常量池索引
	u2 descriptor_index;				// 指向该字段或方法 描述符常量项 的常量池索引
}

CONSTANT_MethodHandle_info {
	u1 tag;								// 标志位，必须为 15
	u1 reference_kind_value;			// 值区间必须=为[1, 9]，决定了方法句柄的类型(kind)，方法句柄类型的值表示方法句柄的字节码行为
	u2 reference_index;					// 根据 reference_kind_value 选择常量池项的类型
}

CONSTANT_MethodType_info {
	u1 tag;								// 标志位，必须为 16
	u2 descriptor_index;				// 方法的描述符，值必须是对常量池的有效索引，常量池在该索引处的类型必须是 CONSTANT_Utf8_info 结构
}

CONSTANT_Dynamic_info {
	u1 tag;								// 标志位，必须为 17
	u2 bootstrap_method_attr_index;		// 值必须是对当前Class文件中引导方法表的bootstrap_methods[]数组的有效索引
	u2 name_and_type_index;				// 指向方法名和方法描述符 CONSTANT_NameAndType_info 的常量池索引
}

CONSTANT_InvokeDynamic_info {
	u1 tag;								// 标志位，必须为 18
	u2 bootstrap_method_attr_index;		// 值必须是对当前Class文件中引导方法表的bootstrap_methods[]数组的有效索引
	u2 name_and_type_index;				// 指向方法名和方法描述符 CONSTANT_NameAndType_info 的常量池索引
}

CONSTANT_Module_info {
	u1 tag;								// 标志位，必须为 19
	u2 module_name_index;				// 模块名称，必须是对常量池的有效索引，常量池在该项的类型必须是 CONSTANT_Utf8_info 结构
}

CONSTANT_Package_info {
	u1 tag;								// 标志位，必须为 20
	u2 package_name_index;				// 包的名称，必须是对常量池的有效索引，常量池在该项的类型必须是 CONSTANT_Utf8_info 结构
}
```

> TODO：对 CONSTANT_MethodHandle_info 的 reference_index 取值进行更详细的介绍


### 3.3 常量池项解析举例


本节将以 CONSTANT_Utf8_info 和 CONSTANT_Float_info 为例，说明该程序对常量池项的解析过程。


--------


当 `tag == 1` 时，表示 UTF8 信息，其结构为：


```
CONSTANT_Utf8_info {
	u1 tag;								// 标志位，必须为 1
	u2 utf8_string_length;				// Utf8 编码的字符串的长度
	utf8_string_length utf8_string;		// Utf8 编码的字符串
}
```


解析时，会通过如下的 Utf8Const 类：


```

import vm.clazz.Util;

/**
 * 常量池信息 - UTF8 类型
 */
public final class Utf8Const extends ConstInfo {


    /** Utf8编码的字符串的长度 u2 */
    private final byte[] length;
    /** Utf8编码的字符串 */
    private final byte[] utf8String;

    /**
     * 构造方法
     *
     * @param content 字节数组
     * @param pos     当前读取的位置
     * @param tag     常量池 tag 位
     */
    public Utf8Const(byte[] content, int pos, int tag) {
        super(tag);

        length = new byte[U2];
        pos += Util.read(content, pos, length);

        utf8String = new byte[Util.unsignedBytes2Int(length)];
        pos += Util.read(content, pos, utf8String);
    }

    @Override
    public int getTag() {
        return TAG_UTF8;
    }

    @Override
    public String getType() {
        return TYPE_UTF8;
    }

    @Override
    public long getOffset() {
        return length.length + utf8String.length;
    }
	
	// ...

}
```

------------


当 `tag == 4` 时，表示 Float 信息，其结构为：


```java
CONSTANT_Float_info {
	u1 tag;								// 标志位，必须为 4
	u4 float_value;						// Float 类型的数值表示
}
```


解析时，会通过如下的 FloatConst 类：


```

import vm.clazz.UnsignedByte;
import vm.clazz.Util;

/**
 * 常量池信息 - Float 类型
 */
public final class FloatConst extends ConstInfo {

    /** Float类型的数值表示 */
    final byte[] floatValue;

    public FloatConst(byte[] content, int pos, int tagValue) {
        super(tagValue);

        floatValue = new byte[U4];
        pos += Util.read(content, pos, floatValue);
    }

    @Override
    public int getTag() {
        return TAG_FLOAT;
    }

    @Override
    public String getType() {
        return TYPE_FLOAT;
    }

    @Override
    public long getOffset() {
        return floatValue.length;
    }


	// ...
}
```


## 4 读取 接口/字段/方法 列表(interfaces/fields/methods)


读取接口列表的步骤分为以下几步：
- 读取接口列表的长度
- 根据接口列表的长度，依次读取每个接口表


```
/**
 * 读取接口列表
 */
private void readInterfaces() {
    int interfacesCountValue = (int) valueOf(interfacesCount);
    interfaces = new InterfaceTable[interfacesCountValue];
    for (int i = 0; i < interfaces.length; i++) {
        interfaces[i] = new InterfaceTable(content, pos);
        pos += interfaces[i].getOffset();
    }
}
```


每个接口表的内容其实只有表示接口名称的常量池索引：


```
interface_info {
	u2 interface_name_index;		// 接口名称的常量池索引
}
```


其读取与常量池项的读取类似，程序如下所示：


```
import vm.clazz.IConstPoolInfo;
import vm.clazz.Util;
import vm.clazz.cp.SeqRead;

/**
 * 接口表
 */
public class InterfaceTable implements IConstPoolInfo, SeqRead {

    /** 接口名称的常量池索引 */
    final byte[] nameIndex;

    /**
     * 构造方法
     *
     * @param content 字节数组
     * @param pos     当前读取的位置
     */
    public InterfaceTable(byte[] content, int pos) {
        nameIndex = new byte[U2];
        pos += Util.read(content, pos, nameIndex);
    }

    @Override
    public long getOffset() {
        return nameIndex.length;
    }
}
```



--------



读取字段列表的步骤分为以下几步：
- 读取字段列表的长度
- 根据字段列表的长度，依次读取每个字段表


每个字段表的内容包含以下几项：


```
field_info {
	u2 access_flag;				// 访问标志
	u2 field_name_index; 		// 名称 - 常量池索引
	u2 descriptor_index;		// 描述符 - 常量池索引
	u2 attribute_count;			// 属性列表的长度
	attribute_count attributes;	// 属性列表
}
```


--------


读取方法列表的步骤分为以下几步：
- 读取方法列表的长度
- 根据方法列表的长度，依次读取每个方法表


每个方法表的内容包含以下几项：


```
method_info {
	u2 access_flag;				// 访问标志
	u2 method_name_index; 		// 名称 - 常量池索引
	u2 descriptor_index;		// 描述符 - 常量池索引
	u2 attribute_count;			// 属性列表的长度
	attribute_count attributes;	// 属性列表
}
```


## 5 读取属性列表(attributes)

属性表的通用格式如下：


```
attribute_info {
	u2 attribute_name_index;	// 属性名称的常量池索引
	u4 attribute_length;		// 字节形式的属性内容的字节数
	attribute_length info;		// 字节形式的属性内容
}
```


其解析程序如下：


```java
import vm.clazz.IConstPoolInfo;
import vm.clazz.Util;
import vm.clazz.cp.SeqRead;

/**
 * 属性表 (attribute_info)
 */
public class Attr implements SeqRead, IConstPoolInfo {
    /** u2 属性名称 - 常量池索引 */
    private final byte[] attributeNameIndex;
    /** u4 属性值所占用的位数 */
    private final byte[] attributeLength;
    /** u1 属性的内容，长度为 attributeLength */
    private final byte[] info;

    public Attr(byte[] source, int sourcePos) {
        attributeNameIndex = new byte[U2];
        sourcePos += Util.read(source, sourcePos, attributeNameIndex);

        attributeLength = new byte[U4];
        sourcePos += Util.read(source, sourcePos, attributeLength);

        info = new byte[Util.unsignedBytes2Int(attributeLength)];
        sourcePos += Util.read(source, sourcePos, info);
    }

    @Override
    public long getOffset() {
        return attributeNameIndex.length + attributeLength.length + info.length;
    }
}
```


而更具体地，可以将attribute分为以下类型：



属性名称                                |  使用位置                             |           含义
-------------------------------------  | ------------------------------------ | ------------------------------------------------------------------------------------------
Code                                   | 方法表                               | Java 代码编译成的字节码指令
ConstantValue                          | 字段表                               | 由 final 关键字定义的常量值
Deprecated                             | 类、方法表、字段表                    | 被声明为 deprecated 的类、方法和字段
Exceptions                             | 方法表                               | 方法抛出的异常列表
EnclosingMethod                        | 类文件                               | 仅当一个类为局部类或者匿名类时才能拥有这个属性，用于标识这个类所在的外围方法
InnerClasses                           | 类文件                               | 内部类列表
LineNumberTable                        | Code 属性                            | Java 源码的行号与字节码指令的对应关系
LocalVariableTable                     | Code 属性                            | 方法的局部变量表描述
StackMapTable                          | Code 属性                            | (JDK 6)提供新的类型检查验证器检查和处理目标方法的局部变量和操作数栈所需要的类型是否匹配
Signature                              | 类、方法表、字段表                    | (JDK 5)用于支持泛型情况下的方法签名，用于记录泛型中的相关信息
SourceFile                             | 类                                   | 记录源文件名称
SourceDebugExtension                   | 类文件                               | (JDK 5)用于存储额外的调试信息。譬如在进行JSP调试时，无法通过Java堆栈来定位到JSP的行号，JSR 45提案为这些非Java语言、却需要编译成字节码并运行在Java虚拟机中的程序提供了一个进行调试的标准机制，用于存储调试信息
Synthetic                              | 类、方法表、字段表                    | 标识方法或字段为编译器自动生成的
LocalVariableTypeTable                 | 类                                   | (JDK 5)它使用特征签名代替描述符，是为了引入泛型语法之后能描述泛型参数化类型而添加
RuntimeVisibleAnnotations              | 类、方法表、字段表                    | (JDK 5)为动态注解提供支持。该属性用于指明哪些注解是运行时(实际运行时就是进行反射调用)可见的
RuntimeInvisibleAnnotations            | 类、方法表、字段表                    | (JDK 5)与RuntimeVisibleAnnotations刚好相反，指明哪些注解是运行时不可见的
RuntimeVisibleParameterAnnotations     | 方法表                               | (JDK 5)作用与RuntimeVisibleAnnotations类似，只不过作用对象为方法参数
RuntimeInvisibleParameterAnnotations   | 方法表                               | (JDK 5)作用与RuntimeVisibleParameterAnnotations相反
AnnotationDefault                      | 方法表                               | (JDK 5)用于记录注解类元素的默认值
BootstrapMethods                       | 类文件                               | (JDK 7)用于保存invokedynamic指令引用的引导方法限定符
RuntimeVisibleTypeAnnotations          | 类、方法表、字段表、Code 属性         | (JDK 8)为实现JSR 308中新增的类型注解提供的支持，指明哪些类注解是运行时可见的
RuntimeInvisibleTypeAnnotations        | 类、方法表、字段表、Code 属性         | (JDK 8)与RuntimeVisibleTypeAnnotations属性刚好相反
MethodParameters                       | 方法表                               | (JDK 8)用于支持(加上-parameters参数)将方法参数名称编译进Class文件中，并可运行时获取
Module                                 | 类                                   | (JDK 9)用于记录一个Module的名称以及相关信息(requires、exports、opens、uses、provides)
ModulePackages                         | 类                                   | (JDK 9)用于记录一个模块中所有被export或者opens的包
ModuleMainClass                        | 类                                   | (JDK 9)用于指定一个模块的主类
NestHost                               | 类                                   | (JDK 11)用于支持嵌套类(Java中的内部类)的反射和访问控制的API,一个内部类通过该属性得知自己的宿主类
NestMembers                            | 类                                   | (JDK 11)用于支持嵌套类(Java中的内部类)的反射和访问控制的API,一个宿主类通过该属性得知自己有哪些内部类


> TODO：解析出更具体的属性(例如Code、ConstantValue等)




## 6 常量池项的格式化


### 6.1 Utf8 的格式化


即使含有中文字符，也可以通过以下方法将字节数组转换成字符串形式：


```
public static String toUtf8String(byte[] utf8StringBytes) {
    return new String(utf8StringBytes);
}
```


### 6.2 Integer / Long 的格式化


针对 CONSTANT_Integer_info 类型，首先对无符号字节数组进行处理，转换成short，再按照 Integer 在二进制中的存储形式，转成十进制形式。


```java
public static int toInt(byte[] integerValueBytes) {
    UnsignedByte[] ubs = UnsignedByte.from(integerValueBytes);
    return (ubs[0].getValue() << (8 * 3))
            + (ubs[1].getValue() << (8 * 2))
            + (ubs[2].getValue() << (8 * 1))
            + (ubs[3].getValue() << (8 * 0));
}
```


CONSTANT_Long_info 类型的处理方式与 CONSTANT_Integer_info 类似：


```java
public static long toLong(byte[] longValueBytes) {
    UnsignedByte[] bytesUB = UnsignedByte.from(longValueBytes);
    return (bytesUB[0].getValue() << (8 * 7))
            + (bytesUB[1].getValue() << ((8 * 6)))
            + (bytesUB[2].getValue() << ((8 * 5)))
            + (bytesUB[3].getValue() << (8 * 4))
            + (bytesUB[4].getValue() << (8 * 3))
            + (bytesUB[5].getValue() << (8 * 2))
            + (bytesUB[6].getValue() << (8 * 1))
            + bytesUB[7].getValue();
}
```


### 6.3 Float / Double 的格式化


要解决 Float / Double 格式化的问题，首先要理解 float 和 double 在二进制中存储的格式：


|| 符号位 | 阶码 | 尾数 | 长度 |
| ---- | ---- | ---- | ---- |
| float  |   1  |   8 |   23 | 32 |
| double |   1 |   11 |   52   | 64 |


---


下面是一个将 float 二进制形式转为十进制形式的例子：


`-25.125f` 的二进制形式：


```
11000001 11001001 00000000 00000000
```


按照 符号位-阶码-尾数 分隔为：


```
1  10000011  10010010000000000000000
```
- 其中 1 表示该浮点数为负数；
- 10000011 的十进制为：128 + 2 + 1 = 131，减去127((1 << 7) - 1)得到4，表示小数点右移的数位
- 剩下 23 位是纯二进制小数
	- 将10010010000000000000000表示成小数，为0.10010010000000000000000
	- 前面加1得到1.10010010000000000000000
	- 小数点右移4位得到11001.0010000000000000000
	- 变为10进制，得到 (1 + 8 + 16).(1/8) = 25.125


前面的1表示负数，所以最终该浮点数的十进制表示为 `-25.125`




----


针对 CONSTANT_Float_info 类型，将其无符号字节数组转换成十进制数值表示的方法如下：


```
/**
 * 将二进制的float转换成十进制数值 
 *
 * @param floatValueBytes 4字节
 * @return floatValueBytes的十进制数值表示
 */
public static float parseFloat(byte[] floatValueBytes) {
    int data = (int) UnsignedByte.valueOf(floatValueBytes);
    boolean positive = ((data & 0b10000000_00000000_00000000_00000000) >> 31) == 0; // 第1位
    int exponent = ((data & 0b01111111_10000000_00000000_00000000) >> 23) - 127; // 阶码
    int mantissa = (data & 0b00000000_11111111_11111111_11111111); // & 0b00000000_01111111_11111111_11111111 后，首位"+1"

    int left = mantissa >> (23 - exponent);
    int right = mantissa << (32 - 23 + exponent) >> (32 - 23 + exponent);

    int dividend = calcDividend(right, 23 - exponent);
    float floatValue = (left + 1.0f / dividend);
    return positive ? floatValue : -floatValue;
}

/**
 * 计算小数点后的二进制的十进制表示，如.001表示8，再用1/8得到0.125
 */
private static int calcDividend(int rightBits, int len) {
    int result = 0;
    while (rightBits != 0) {
        int lastBits = rightBits & 1; // 最后一位是1，则结果为1，否则为0
        result += lastBits * Math.pow(2, len);
        rightBits >>= 1;
        len--;
    }
    return result;
}
```


----


CONSTANT_Double_info 与 CONSTANT_Float_info 类似。



## 7 TODO事项


- 对 CONSTANT_MethodHandle_info 的 reference_index 取值进行更详细的介绍
- 解析出更具体的属性(例如Code、ConstantValue等)
- 扩展到dex文件结构的解析