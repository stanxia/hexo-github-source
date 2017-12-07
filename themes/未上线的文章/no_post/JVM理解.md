---
title: JVM理解
date: 2016-03-01 14:06:57
tags: JVM
categories: java
---

## 什么是jvm

jvm（java virtual machine）,java虚拟机，就是在计算机的内存中再虚拟出个计算机。
为什么java一次编译，随处运行？
都归功于jvm的命令集，jvm翻译之后，根据不同的cpu，翻译成不同的机器语言。

<!-- more -->

## jvm的组成部分

1. ##### Class Loader 类加载器

   将javac 编译后的 .class文件加载到内存中。（注意：并不会加载所有的.class文件，而是只加载符合class规范的文件。可阅读<JVM Specification>中的第四章"The Class File Format"）

2. ##### Execution Engine 执行引擎

   也叫做解释器(Interpreter)，负责解释命令，提交操作系统执行。

3. ##### Native Interface 本地接口

   本地接口的作用是融合不同的语言为java所用。

4. ##### Runtime Data Area 运行数据区

   所有程序被加载到运行数据区域才能运行。

![1](http://wx1.sinaimg.cn/mw1024/6aae3cf3gy1fd7kltd18vj20ye0leq7j.jpg)



## jvm 的内存管理

所有的程序都是被加载到运行数据区域才能执行。
运行数据区主要包括：

1. ##### Stack 栈

   栈也叫做栈内存，与线程同生死。线程创建时创建，线程结束时自动释放栈内存，不需要GC。
   栈的原则：先进后出。
   栈中存放的数据格式：栈帧(Stack Frame)
   栈帧：方法和运行期数据的数据集
   栈帧包括：
   a. 本地变量(local variables)，包括输入，输出参数以及方法内的变量
   b. 栈操作(Operand Stack)，记录进出栈的操作
   c. 栈帧数据(Frame Data),包括类文件，方法等

   ​

   ![2](http://wx1.sinaimg.cn/mw1024/6aae3cf3gy1fd7kltq5vcj20f90oi0u1.jpg)

   java 栈结构图 

   图示：栈中有两个栈帧，栈帧2是最先被调用的方法，先入栈。方法2又调用方法1，栈帧1入栈，且位于栈顶，栈帧2位于栈底。执行完成后，先弹出栈帧1，再弹出栈帧2.线程结束，释放栈。

2. ##### Heap 堆内存

   一个jvm只有一个堆内存，大小可调节。类加载器加载了类文件后，需要把类，方法，常变量放到堆内存中，以方便解释器执行。
   堆内存结构：
   a. Permanent Space永久存储区
   永久存储区是一个常驻内存区域。用于存放jdk自身所携带的Class,Interface的元数据，即存储的是运行环境所必须的类信息，该区域中的数据不会被GC回收，只有关闭jvm才会释放该区域所占内存。
   b. Young Genaration Space 新生区
   新生区是类的诞生，成长，消亡的区域。一个类在这里产生，应用，最后被GC收回。
   新生区分为两个区：伊甸区(Eden Space)和幸存者区(Survivor space)。所有类都在伊甸区被new出来的。
   幸存区有两个：0区(Survivor 0 space)和1区(Survivor 1 space)。
   当伊甸区的空间用完时，程序有需要new新的类，这是GC将对伊甸区进行回收，将伊甸区中不再被引用的对象进行销毁，然后将伊甸区中剩余的对象移动到幸存0区。若幸存0区也满了，在对该区域进行垃圾回收，然后将剩余的移动到1区。如果1区也满了，再移动到养老区。
   c. Tenure Generation Space 养老区
   养老区用于保存被新生区筛选出来的对象。一般池对象都保存在这个区。

   ![3](http://wx2.sinaimg.cn/mw1024/6aae3cf3gy1fd7klu48owj20ci09mgo7.jpg)

3. ##### Method Area 方法区

   方法区是被所有线程共享，该区域保存所有字段和方法字节码，以及一些特殊方法如 构造函数，接口代码也在此定义。

4. ##### Program Counter Register 程序计数器

   每个线程都有一个程序计数器，就是一个指针，指向方法区中的方法字节码，由执行引擎读取下一条指令。

5. ##### Native Method Stack 本地方法栈

## JVM相关问题

##### 堆和栈的区别？

1. 堆中存放对象，对象内的临时变量存放在栈中。
2. 栈随线程同生死，堆随 jvm同生死。

##### 堆内存存放什么？

对象，包括对象变量以及对象方法。

##### 类变量和实例变量的区别？

1. 静态变量是类变量，非静态变量是实例变量。

2. 静态变量存放在方法区中，实例变量存放在堆内存中。


##### 为什么产生OutOfMemory?

Heap堆内存中没有足够的内存可用。新申请的内存大于堆中的空闲内存。

##### 产生的对象不多，为什么也会出现OutOfMemory?

继承的层次太多，Heap堆内存中产生对象是先产生父类，然后才产生子类。

