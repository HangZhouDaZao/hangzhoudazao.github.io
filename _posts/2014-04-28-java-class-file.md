---
layout: post
title: "java class file和一个反汇编器（javap）的实现"
description: ""
category: "Java"
tags: [Java]
---
{% include JB/setup %}

本文未完，还需要加入

```
1. 一个具体的实例讲解
2. 反汇编器讲解
```

Java这语言由于个人莫名其妙的原因，我在工作前是尽量不用的，哪怕是在大学时有门Java编程相关的课程，也是水水过去。“尽量不用”的结果就是不会用。当然写简单的程序，修改修改代码，对于有编程基础的人来说，Java实在是太简单了，马上就能上手。但是对于强迫症患者来说，这样是很不爽的。虽然我觉得目前我面对Java时最大的困难是难以一下子掌握他庞大的库[^1]，但是我还是决定从基础学起，哪怕这对真正工作时帮不了什么忙。


[^1]: 这里说库可能不太准确，Java现成的东西太多了，比如各种各样的框架啊，那些东西熟悉起来真的是略多，关键还没意思。上次和hgfeaon谈起的时候，他的评论：Java实在太简单了，导致了那帮写Java的人太闲了，弄出了各种各样的框架。

讲Java语法的书我是懒得看了，哪怕是《thinking in java》我也懒得看。以前看过《thinking in C++》，然后结果就是忘光了，于是我明白了一个我明白了多次的道理：看语法书和经验之谈在没有实际编码经历之前都是没用的。正好这时候看到CC98上的BIAOBIAO齐“学姐”推荐[深入理解Java虚拟机](http://book.douban.com/subject/6522893/)作为入门教材。鉴于我对BIAOBIAO齐“学姐”的靠谱印象，于是我就去翻了几页，然后翻着翻着就翻到了[Java虚拟机规范](http://docs.oracle.com/javase/specs/jvms/se7/jvms7.pdf)，然后就准备写一系列博客了，对，这就是第一篇，希望不是最后一篇。

和此博客相关的是我在学习过程中写的一个小程序，他是javap[^2]的一个克隆[javap](https://github.com/fengya90/javatoy/tree/master/javap)。我觉得学习java class file最好还是阅读Java虚拟机规范然后动手实现一些相关工具。

[^2]: javap是jdk中带的一个反汇编器

## The class File Format

Java class file说白了就是一种二进制文件，可以和ELF文件类比。只不过[ELF](http://en.wikipedia.org/wiki/Executable_and_Linkable_Format)文件由Linux内核和动态链接器（一般为ld.so）合作加载、解析和执行，而Java class file则由java虚拟机来完成这些动作。如果以Java虚拟机作为标准制造CPU并完成相关操作系统开发，那么Java class file和ELF文件就是同一个地位了。相对而言，Java class file比ELF简单的多，主要原因在于Java标准只有一个，而ELF却要支持x86、PPC、MIPS等多种CPU架构。

### 文件格式总览

#### 一些约定

The Java® Virtual Machine Specification里的约定，把一个字节（8位数据）记作u1，两个字节记作u2,4个字节记作u4，以此类推。

一下基本参照The Java® Virtual Machine Specification。

#### 总体格式

 <pre class="prettyprint">
   ClassFile {
       u4             magic;
       u2             minor_version;
       u2             major_version;
       u2             constant_pool_count;
       cp_info        constant_pool[constant_pool_count-1];
       u2             access_flags;
       u2             this_class;
       u2             super_class;
       u2             interfaces_count;
       u2             interfaces[interfaces_count];
       u2             fields_count;
       field_info     fields[fields_count];
       u2             methods_count;
       method_info    methods[methods_count];
       u2             attributes_count;
       attribute_info attributes[attributes_count];
}
</pre>

magic用于判断文件类型，Java的magic是0xCAFEBABE。很容易看出，这个magic只是有指导意义，任何文件都可以添加这个maigc，到底是不是Java class file，还要进一步验证。

major_version和minor_version，顾名思义为大小版本号，没啥好说的。

constant_pool_count就是constant_pool的count，他用来表明constant_pool中有几项。而constant_pool中则是一系列的结构，每种结构大小是不一样的，但是这里称他们为cp_info。constant_pool的每条记录都是有序号的，序号从1到constant_pool_count-1。没错，不是从0开始的，所以constant_pool里有constant_pool_count-1而不是constant_pool_count条记录。

access_flags中记录的是class file中类的访问权限，比如public private等。

this_class中记录的是一个constant_pool索引，通过这个索引和constant_pool可以获得这个类的名字。super_class类似，只不过代表的是父类。

interfaces_count表示这个类实现的接口，interfaces中每一项都是constant_pool中的索引，根据interfaces_count、interfaces和constant_pool就能确认各个接口的名字。

下面的fields_count、methods_count和attributes_count分别代表fields、methods和attributes中的项数，至于fields、methods和attributes这三种结构，则参见下文。


### The Constant Pool

Constant Pool里的每一项的一般结构如下

 <pre class="prettyprint">
	cp_info {
		u1 tag;
		u1 info[];
	}
</pre>


tag用来表示这个结构是什么数据，比如如果tag的值为1，则这个结构的类型为CONSTANT_Utf8，即utf8编码的字符串，如果tag为7，表示这个结构体是CONSTANT_Class。
Constant Pool有好几种类型，这里举几个例子。
#### CONSTANT_Utf8_info

此结构如下所示：

<pre class="prettyprint">
	cp_info {
		u1 tag;
		u2 length;
		u1 bytes[length];
	}
</pre>

tag的值表明了这是个CONSTANT_Utf8_info类型的结构，length用来表明后面还有几个bytes，实际就是用于解析过程中判断结构大小。然后是bytes，里面的内容为utf8编码的字符串。文件名啊、类名啊最后就存在这些结构体中。

#### CONSTANT_Class_info
此结构用来表示类或者接口，结构如下：

<pre class="prettyprint">
  CONSTANT_Class_info {
       u1 tag;
       u2 name_index;
   }
</pre>

tag不必多说，而name_index实际上表示的是Constant Pool中的一个索引，其指向的的结构是CONSTANT_Utf8_info，也就是字符串。
上面说的this_class指向的实际上就是CONSTANT_Class_info，然后通过CONSTANT_Class_info可以找出CONSTANT_Utf8_info，获得类名[^3]。
[^3]: 没明白为什么要多走一步，this_class直接指向CONSTANT_Utf8_info不就好了

### Attributes

虽然上面结构总览中看到Fields和Methods先于Attributes，但是实际上Fields和Methods中包含有Attributes，所以我打算先写Attributes。

Attributes在ClassFile、field_info、method_info以及Attributes自身中都有出现。Attributes基本结构如下：

<pre class="prettyprint">
	attribute_info {
       u2 attribute_name_index;
       u4 attribute_length;
       u1 info[attribute_length];
	}
</pre>

attribute_name_index是Constant Pool中的索引，内容是个字符串，根据字符串来判断Attributes的类别。attribute_length用来指示info有多少个，其实就是确定结构大小。

因为Attributes有好多种类别，和上面一样，这里也举几个例子

#### SourceFile
SourceFile顾名思义，用来记录源文件信息
结构如下

<pre class="prettyprint">
SourceFile_attribute {
       u2 attribute_name_index;
       u4 attribute_length;
       u2 sourcefile_index;
}
</pre>

明显的，attribute_length的值只能等于2了，sourcefile_index是Constant Pool中的索引，存的是源文件名字符串
#### LocalVariableTable

LocalVariableTable存在于Code（见下文）里，用于描述栈帧中局部变量和源代码的关系，主要用于调试。

结构如下

<pre class="prettyprint">
   LocalVariableTable_attribute {
       u2 attribute_name_index;
       u4 attribute_length;
       u2 local_variable_table_length;
       {   u2 start_pc;
           u2 length;
           u2 name_index;
           u2 descriptor_index;
           u2 index;
       } local_variable_table[local_variable_table_length];
   }
</pre>

local_variable_table里的各项解释我就直接引用了





#### Code
结构如下

<pre class="prettyprint">
   Code_attribute {
       u2 attribute_name_index;
       u4 attribute_length;
       u2 max_stack;
       u2 max_locals;
       u4 code_length;
       u1 code[code_length];
       u2 exception_table_length;
       {   u2 start_pc;
           u2 end_pc;
           u2 handler_pc;
           u2 catch_type;
       } exception_table[exception_table_length];
       u2 attributes_count;
       attribute_info attributes[attributes_count];
      }
</pre>

### Fields和Methods

Fields和Methods放在一起讲，是因为他们结构基本一样。

<pre class="prettyprint">
  field_info {
       u2             access_flags;
       u2             name_index;
       u2             descriptor_index;
       u2             attributes_count;
       attribute_info attributes[attributes_count];
}

   method_info {
       u2             access_flags;
       u2             name_index;
       u2             descriptor_index;
       u2             attributes_count;
       attribute_info attributes[attributes_count];
}
</pre>

access_flags记录权限控制，name_index和常量池确定field或者method的名字。descriptor_index和常量池确定field的类型或者method的返回类型。attributes_count和attributes则记录一系列的返回值。



## 反汇编器的实现
### 反汇编流程


### 数据结构




## 参考文献：

* [Java虚拟机规范](http://docs.oracle.com/javase/specs/jvms/se7/jvms7.pdf)
* [深入理解Java虚拟机](http://book.douban.com/subject/6522893/)
