---
title: Java IO -- IO流的概念和分类
categories: Java
comments: true
tags: [Java]
description: 介绍IO流的概念和分类
date: 2014-4-12 10:00:00
---

## 概述

Java 的I/O流是实现输入/输出的基础，它可以方便地实现数据的输入/输出操作，在 Java 中把不同的输入/输出源（键盘、文件、网络等）抽象表示为流。这样做简化了输入/输出处理。
Java 的IO流涉及到了40多个类，如下图所示：

![效果图](/images/java-io-stream-base/io-class.png)

## I/O流的分类

### 字符流和字节流

I/O 按照处理数据类型分为：字符流（Reader和Writer及其子类）和字节流（InputStream和outPutStream及其子类）。
字符流和字节流的用法几乎是一样的，它们的区别如下：

 - 操作数据单元不同：字节流是以字节（8bit）为单位，字符流是以字符为单位，根据码表映射字符，一次可能读多个字节。
 - 处理对象不同：字节流能处理所有类型的数据（如图片、avi等），而字符流只能处理字符类型的数据。
 - 字节流在操作的时候本身是不会用到缓冲区的，是文件本身的直接操作的；而字符流在操作的时候下后是会用到缓冲区的，是通过缓冲区来操作文件。

我们平时使用时优先选择使用字节流，首先因为硬盘上的所有文件都是以字节的形式进行传输或者保存的，包括图片等内容。但是字符只是在内存中才会形成的，所以在开发中，字节流使用广泛。

### 输入流和输出流

对输入流只能进行读操作（InputStream和Reader以及它们的子类），对输出流只能进行写操作（OutputStream和Writer以及它们的子类）。

### 节点流和处理流

按照流的角色来分，可以分为节点流和处理流。处理流也叫过滤流和包装流。
使用节点流进行输入/输出时，程序直接连接到实际的数据流，和实际的输入/输出节点连接。节点流也称为低级流。
处理流则用于对一个已存在的流进行连接或封装，通过封装后的流来实现数据读写功能，可以更加灵活方便地读写各种类型的数据。处理流也被称为高级流。
当使用处理流进行输入/输出时，程序并不会直接连接到实际的数据源，没有和实际的输入/输出节点连接。使用处理流的一个明显好处是，只要使用相同的处理流，程序就可以采用完全相同的输入/输出代码类访问不同的数据源，随着处理流所包装的节点的变化，程序实际所访问的数据源也相应地发生变化。
FilterInputStream、FilterOutputStream、FilterReader和FilterWriter以及它们的子类是处理流。
实际上是通过装饰器模式来使用处理流来包装节点流，通过包装不同的节点流，即可以消除不同节点流的实现差异，也可以提供更方便的方法来完成输入/输出功能，因此处理流也称为包装流。

#### 节点流

| 类型 | 字符流 | 字节流 |
|:-------------:|:-------------:|:-------------:|
| File | FileReader、FileWriter | FileInputStream、FileOutputStream |
| Memory Array | CharArrayReader、CharArrayWriter | ByteArrayInputStream、ByteArrayOutputStream |
| Memory String | StringReader、StringWriter | StringBufferInputStream、 |
| Pipe | PipedReader、PipedWriter | PipedInputStream、PipedOutputStream |

#### 处理流

| 类型 | 字符流 | 字节流 |
|:-------------:|:-------------:|:-------------:|
| Buffer | BufferedReader、BufferedWriter | BufferedInputStream、BufferedOutputStream |
| Filter | FilterReader、FilterWriter | FilterInputStream、FilterOutputStream |
| Converter | InputStreamReader、OutputStreamWriter |  |
| Object Serialization |  | ObjectInputStream、ObjectOutputStream |
| DataConversion |  | DataInputStream、DataOutputStream |
| Counting | LineNumberReader | LineNumberInputStream |
| Peeking Ahead | PushbackReader | PushbackInputStream |
| Printing | PrintWriter | PrintStream |


 - Buffering缓冲流：在读入或写出时，对数据进行缓存，以减少I/O的次数：BufferedReader与BufferedWriter、BufferedInputStream与BufferedOutputStream。
 - Filtering 滤流：在数据进行读或写时进行过滤：FilterReader与FilterWriter、FilterInputStream与FilterOutputStream。
 - Converting between Bytes and Characters 转换流：按照一定的编码/解码标准将字节流转换为字符流，或进行反向转换（Stream到Reader）：InputStreamReader、OutputStreamWriter。
 - Object Serialization 对象流：ObjectInputStream、ObjectOutputStream。
 - DataConversion数据流：按基本数据类型读、写（处理的数据是Java的基本类型（如布尔型，字节，整数和浮点数））：DataInputStream、DataOutputStream 。
 - Counting计数流：在读入数据时对行记数 ：LineNumberReader、LineNumberInputStream。
 - Peeking Ahead预读流：通过缓存机制，进行预读：PushbackReader、PushbackInputStream。
 - Printing打印流：包含方便的打印方法：PrintWriter、PrintStream。 


#### 缓冲流

对I/O进行缓冲是一种常见的性能优化，缓冲流为I/O流增加了内存缓冲区，增加缓冲区的两个目的： 

 - 允许Java的I/O一次不只操作一个字符，这样提高䇖整个系统的性能
 - 由于有缓冲区，使得在流上执行skip、mark和reset方法都成为可能。

缓冲流：它是要“套接”在相应的节点流之上，对读写的数据提供了缓冲的功能，提高了读写的效率，同时增加了一些新的方法。例如：BufferedReader中的readLine方法，BufferedWriter中的newLine方法。
缓冲输入流BufferedInputSTream除了支持read和skip方法意外，还支持其父类的mark和reset方法;
BufferedReader提供了一种新的ReadLine方法用于读取一行字符串（以\r或\n分隔）;
BufferedWriter提供了一种新的newLine方法用于写入一个行分隔符;
对于输出的缓冲流，BufferedWriter和BufferedOutputStream，写出的数据会先在内存中缓存，使用 flush 方法将会使内存的数据立刻写出。





<!-- 
@startuml
Title "继承关系图"

字节流 -> InputStream
字节流 -> OutputStream

abstract class OutputStream
class FileOutputStream
class FilterOutputStream
class ObjectOutputStream
class PipedOutputStream
class ByteArrayOutputStream

class BufferedOutputStream
class DataOutputStream
class PrintStream

OutputStream <|-- FileOutputStream
OutputStream <|-- FilterOutputStream
OutputStream <|-- ObjectOutputStream
OutputStream <|-- PipedOutputStream
OutputStream <|-- ByteArrayOutputStream

FilterOutputStream <|-- BufferedOutputStream
FilterOutputStream <|-- DataOutputStream
FilterOutputStream <|-- PrintStream


abstract class InputStream
class FileInputStream
class FilterInputStream
class ObjectInputStream
class PipedInputStream
class SequenceInputStream
class StringBufferInputStream
class ByteArrayInputStream

class BufferedInputStream
class DataInputStream
class PushbackInputStream

InputStream <|-- FileInputStream
InputStream <|-- FilterInputStream
InputStream <|-- ObjectInputStream
InputStream <|-- PipedInputStream
InputStream <|-- SequenceInputStream
InputStream <|-- StringBufferInputStream
InputStream <|-- ByteArrayInputStream

FilterInputStream <|-- BufferedInputStream
FilterInputStream <|-- DataInputStream
FilterInputStream <|-- PushbackInputStream
@enduml

-->