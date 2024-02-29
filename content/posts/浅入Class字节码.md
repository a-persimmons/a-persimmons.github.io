---
author: ["柿子"]
title: "浅入Class字节码"
date: "2021-06-20 20:59:25"
description: "浅入Class字节码."
summary: "关于Class字节码."
tags: ["Java","JDK","源码"]
---

> 不以规矩，不成方圆

“Write Once，Run Everywhere”，功归于虚拟机的“规范”

### 基本数据类型

- u1、u2、u4分别代表占用1字节、2字节、4字节

### Class文件结构

自上而下

```java
ClassFile {
    u4             magic; // 魔术 0xCAFEBABE
    u2             minor_version;// 次要版本，标识 m
    u2             major_version;// JDK主要版本,标识 M
    u2             constant_pool_count;// 常量池个数
    cp_info        constant_pool[constant_pool_count-1];// 常量池元素数组 下标从1开始
    u2             access_flags;// 此类修饰符
    u2             this_class;// 此包类或接口路径 如 com/amk/App
    u2             super_class;// 父类,至少是Object超类
    u2             interfaces_count;// 实现接口个数
    u2             interfaces[interfaces_count];// 具体的接口名数组
    u2             fields_count;// 属性个数
    field_info     fields[fields_count];// 属性值
    u2             methods_count;// 方法个数
    method_info    methods[methods_count];// 具体方法
    u2             attributes_count;// 其他属性计数
    attribute_info attributes[attributes_count];// 具体其他属性数组
}
```

### 十六进制码

具体看下Class文件

> 需要的工具 Sublime 或者 01Editer
>
> 预设知识：[图解进制转换](https://www.cnblogs.com/gaizai/p/4233780.html)

```java
// 类
public class ClassStructure {}

//javac ClassStructure 编译输出 Class文件
public class ClassStructure {
  // 默认无参构造
  public ClassStructure() {}
}
```

通过 Sublime打开,会看到是这样的,这就是Class 十六进制

```java
cafe babe 0000 0034 0010 0a00 0300 0d07
000e 0700 0f01 0006 3c69 6e69 743e 0100
0328 2956 0100 0443 6f64 6501 000f 4c69
6e65 4e75 6d62 6572 5461 626c 6501 0012
4c6f 6361 6c56 6172 6961 626c 6554 6162
6c65 0100 0474 6869 7301 0010 4c43 6c61
7373 5374 7275 6374 7572 653b 0100 0a53
6f75 7263 6546 696c 6501 0013 436c 6173
7353 7472 7563 7475 7265 2e6a 6176 610c
0004 0005 0100 0e43 6c61 7373 5374 7275
6374 7572 6501 0010 6a61 7661 2f6c 616e
672f 4f62 6a65 6374 0021 0002 0003 0000
0000 0001 0001 0004 0005 0001 0006 0000
002f 0001 0001 0000 0005 2ab7 0001 b100
0000 0200 0700 0000 0600 0100 0000 0400
0800 0000 0c00 0100 0000 0500 0900 0a00
0000 0100 0b00 0000 0200 0c
```

我们看到"**cafe babe**"开头,这个就是**魔数**,可以理解为文件类型描述符,从上文得到魔数数据类型是u4,占用4个字节,得到一个码占用4位,下面我们来尝试读取一下

方法:先看上面的`ClassFile`中的数据类型u*,在看对应的代表什么

- cafebabe : magic u4
- 0000: minor_version u2 **0**
- 0034: major_version u2 **52**=`4x16^0+3x16^1` [表格](https://docs.oracle.com/javase/specs/jvms/se16/html/jvms-4.html#jvms-4.1-200-B.2)
- 0010:constant_pool_count u2 **16**
- ...
- 以此借助基本数据类型读取进制码

接下来看另一种数据类型读取方法: `cp_info`,需要借助IDEA 

- 常量池

  ```shell
  javap -v ClassStructure 
  # 会看到如下结果
  Classfile /target/test-classes/ClassStructure.class
    Last modified 2021年5月9日; size 267 bytes
    SHA-256 checksum d6a39a3bf56d982b11d79c38256f2f6b2a2f8745c38a4f86440fb9d701f44078
    Compiled from "ClassStructure.java"
  public class ClassStructure
    # 次版本
    minor version: 0
    # JDK版本
    major version: 52
    # 类修饰符
    flags: (0x0021) ACC_PUBLIC, ACC_SUPER
    # 类名
    this_class: #2                          // ClassStructure
    # 父类
    super_class: #3                         // java/lang/Object
    # 接口 变量 方法 其他属性 计数
    interfaces: 0, fields: 0, methods: 1, attributes: 1
  # 重点看这儿常量池
  Constant pool:
  # 井号后面是索引
     #1 = Methodref 方法      #3(指向3).#13(指向索引13) // java/lang/Object."<init>":()V
     #2 = Class              #14            // ClassStructure
     #3 = Class              #15            // java/lang/Object
     #4 = Utf8               <init>
     #5 = Utf8               ()V
     #6 = Utf8               Code
     #7 = Utf8               LineNumberTable
     #8 = Utf8               LocalVariableTable
     #9 = Utf8               this
    #10 = Utf8               LClassStructure;
    #11 = Utf8               SourceFile
    #12 = Utf8               ClassStructure.java
    #13 = NameAndType        #4:#5          // "<init>":()V
    #14 = Utf8               ClassStructure
    #15 = Utf8               java/lang/Object
  {
    public ClassStructure();
      descriptor: ()V
      flags: (0x0001) ACC_PUBLIC
      Code:
        stack=1, locals=1, args_size=1
           0: aload_0
           1: invokespecial #1                  // Method java/lang/Object."<init>":()V
           4: return
        LineNumberTable:
          line 4: 0
        LocalVariableTable:
          Start  Length  Slot  Name   Signature
              0       5     0  this   LClassStructure;
  }
  SourceFile: "ClassStructure.java"
  ```
  
- constant_pool_count常量计数后面是常量元素(cp_info),通过规范查到的结构是

  ```java
  cp_info {
      u1 tag;
      u1 info[];
  }
  ```

  所以接下来的`u1`一个字节`0a`为tag `10`,然后在规定的[17个tags](https://docs.oracle.com/javase/specs/jvms/se16/html/jvms-4.html#jvms-4.4-140)得到是"CONSTANT_Methodref"在点击`Section`这一栏的超链接得到对应的结构为

  ```java
  // 先知道是什么,后面有宏观解释
  CONSTANT_Methodref_info {
      u1 tag;
      u2 class_index; // 指向常量池的下一个 类索引 联系上文[Constant pool]#3 
      u2 name_and_type_index; // 入参:返回类型 索引 练习上文[Constant pool]#15
  }
  ```

  宏观解释无参构造方法:

  ```java
  // 联系 javap -v 输出结果 Constant pool
  {
    u1 tag; 0a // 代表17个Tags中tag为10的CONSTANT_Methodref
    // #3->#15->java/lang/Object 表示类为Object 但是实际类型并不是暂时理解为通用类
    // class_index 类为Object占位符,猜想运行时才会转化为具体的类型,TODO 待验证
    u2 class_index; 0003 #3 -> [#3 = Class #15] -> [#15 = Utf8 java/lang/Object]
    // #13->#13 #4name:#5type -> "<init>":()V 表示无参构造方法
    u2 name_and_type_index; 000d #13 -> [#13 = NameAndType #4:#5]-> ["<init>":()V]
  }
  ```

  以上读取了 "0a00 0300 0d"常量池第一个元素的含义,后面的参照此方法

  > 总结读取步骤:
  >
  > - u1 查[tags表 ](https://docs.oracle.com/javase/specs/jvms/se16/html/jvms-4.html#jvms-4.4-140)
  > - 链接Section栏的超链接查看具体的结构字节数
  > - 带着字节数索引在javap -v 总常量池[Constant pool]总向后推进

最后介绍个IDEA插件:[jclasslib Bytecode Viewer](https://plugins.jetbrains.com/plugin/9248-jclasslib-bytecode-viewer)代替Javap -v 更加直观

<img src="https://i.loli.net/2021/08/19/AHqtnewrsENTi5u.png" alt="image-20210509230733460" style="zoom:67%;" />

### 插件参构造表示方式

![image-20210509232109822](https://i.loli.net/2021/08/19/ld9DufrnTHOVwt2.png)

> 参考:[Oracle Class文件结构](https://docs.oracle.com/javase/specs/jvms/se16/html/jvms-4.html#jvms-4.4)

Next：用Java Buffer解析字节码

