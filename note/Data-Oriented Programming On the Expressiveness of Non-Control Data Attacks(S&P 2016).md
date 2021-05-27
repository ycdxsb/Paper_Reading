---
title: >-
  Data-Oriented Programming: On the Expressiveness of Non-Control Data
  Attacks(S&P 2016)
tags:
  - paper
  - security
  - others
author: ycdxsb
categories:
  - papers
  - security
  - others
abbrlink: c30e365e
date: 2020-05-26 16:23:00
---

<!--toc-->


> 根据控制流的攻击我们知道有ROP和JOP，分别利用包含ret和jmp的Gadgets进行攻击，劫持控制流。
>
> 非控制数据攻击通过攻击程序内存，达到信息泄露或者权限提升等目的。文中提出了DOP攻击，利用程序中的Gadgets，构造任意x86程序的非控制数据攻击，并且这种攻击是图灵完备的。

<!--more-->
## Introduction

控制流劫持攻击是目前主流的攻击，例如ROP及其变种，但对此人们也有很多防御措施：CFI、CCFI、CPI、TASR、ASLR、DEP等。

从程序的执行角度，我们可以想到程序是存在控制流和数据流的，而以上只能保证控制流部分的安全，对于数据流则无无能为力，所以非控制数据流攻击就成了额外的攻击方法，只要修改内存中的几个字节，就能达成攻击目的。

**本文方法**：

- 找DOP的gadgets——模拟图灵运算
- 找gadgets dispatcher——串联Gadgets

**实验结果**：

- 9个程序中找到了7518个gadget和5092个gadgets dispatcher
- 其中8个程序能模拟任意计算，2个可以达成图灵完全攻击

最后也实现了3种端到端的攻击，并且ASLR和DEP对攻击无作用。



## Problem

### Background: Non-control Data Attacks

通过直接攻击数据流来达到攻击目的，例如下图中，我们只要修改变量`pw->pw_uid`的值，就能达到提权的目的

![image-20200526132809230](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-07-11-134306.jpg)



### Example of Data-oriented Programming

![image-20200526133329831](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-07-11-134310.jpg)

能看懂啊line 7 存在溢出，因此buf溢出能控制局部变量(tyoe,size,connect_limie)，同时局部变量又能修改指针（line12，line13）这种就称为`data-oriented gadgets`，同时可以注意到它们都在while循环中，因此可以连续的利用，称为`gadget dispatchers`

通过对上图中的DOP利用，能够更新Code3链表中的函数，并且这种攻击时满足CFG完整性的

![image-20200526133757036](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-07-11-134315.jpg)

### Questions

- DOP gadgets和gadgets dispatcher存在普遍吗？
- 能否根据需要链式gadgets达到攻击，是否图灵完备？
- 能否突破当期的防御机制？

## DATA-ORIENTED PROGRAMMING

### DOP Overview

DOP主要是模拟表达式计算，因此定义了如下DOP语言

![image-20200526134248369](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-07-11-134324.jpg)

包括六种虚拟指令，实现算术、逻辑、赋值、加载、存储、跳转、条件跳转等操作。

### Data-Oriented Gadgets

DOP的gadgets不能使用寄存器，使用内存来模拟寄存器。面向数据的gadgets模拟了三种micro-operation：加载，运算和写入。

DOP和ROP很像，他们的区别主要在于以下两点：

1. DOP的gadgets只能使用内存来传递操作的结果，而ROP的gadgets可以使用寄存器。
2. DOP的gadgets必须符合控制流图（CFG），不能发生非法的控制流转移，而且无需一个接一个的执行。而ROP的gadgets必须成链，顺序执行。



**模拟算数运算**：

![image-20200526135232984](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-07-11-134329.jpg)

**模拟赋值运算**：

![image-20200526135300192](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-07-11-134334.jpg)

**模拟加载，存储运算**：

![image-20200526135344429](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-07-11-134339.jpg)

### Gadgets Dispatcher

![image-20200526143825499](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-07-11-134343.jpg)

Dispatcher用来对gadgets进行迭代调度，在每一轮迭代中选用不同的gadgets对上一轮的结果进行处理，为了将第i次迭代的输出和第 i+1 次迭代的输入对应，gadgets将第 i+1 的加载地址设置为第 i 次迭代的存储地址。



除了上述多轮的攻击，还存在一种非交互式的DOP攻击。这种攻击要求攻击者一次性将攻击载荷输入，为了支持这样的攻击，MINDOP中也保留了两个跳转指令，能实现跳转

**模拟跳转**：

![image-20200526144557526](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-07-11-134348.jpg)

关键是找到一个合适的变量，可以在每次循环迭代中修改的虚拟 PC 指针，如上述代码，有一个内存指针 `pubf -> current`，指向了恶意网络输入的缓冲区。在每一次循环迭代中，代码从该缓冲区读取一行，然后在循环体中处理它，因此这个指针可以用来模拟虚拟 PC 指针。对于模拟非条件跳转，攻击者只需要配置好内存，来触发另一个操作 gadgets（如加法、赋值）来改变虚拟 PC 指针的值。



## DOP ATTACK CONSTRUCTION

这里总结一下在DOP过程中需要解决的**三个问题**：

- DOP gadgets识别
- DOP gadgets dispatcher识别
- DOP gadgets的拼接利用，在保证程序不崩溃的前提下进行攻击

### Gadget Identification

一个有用的DOP gadgets需要满足一下**两个条件**：

- 满足MINDOP语义。包含加载、存储、运算指令
- 在顺序上应该满足加载、运算、存储的顺序。

使用LLVM实现对DOP gadgets的识别 (https://github.com/melynx/DOP-StaticAssist)：LLVM IR提供了比二进制更多的程序语义，同时避免了对程序源码的解析。它还允许对任何有LLVM前端的语言编写的源码进行语言诊断分析。

![image-20200526150317786](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-07-11-134354.jpg)

**gadgets分类**：根据语义和运算的变量分为三类，并且在使用优先级上全局变量gadgets>函数参数gadgets>局部变量gadgets

- 全局变量gadgets：操作全局变量
- 函数参数gadgets：操作函数参数
- 局部变量gadgets：操作局部变量

### Dispatcher Identification

同样也基于LLVM IR 实现（https://github.com/melynx/DOP-StaticAssist）

![image-20200526150809559](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-07-11-134359.jpg)

### Attack Contruction

前提：需要攻击者能够控制第一个gadget加载的地址或者第一个gadget存储的地址

**攻击步骤**：

- Gadget preparation (Semi-automated).：根据一个程序错误，定位到漏洞函数，然后找函数中的gadget dispatcher
- Exploit chain construction (Manual)：将预期的恶意 MinDOP 程序输入，每一个 MinDOP 操作由DOP gadgets 实现，并根据优先级选择合适的 gadgets
- Stitchability verification (Manual)：验证是否成功，如果不行回到上一步

## Evaluation

在Evalution中回答了三个问题：

- DOP gadgets和gadgets dispatcher存在普遍吗？
- 能否根据需要链式gadgets达到攻击，是否图灵完备？
- 能否突破当期的防御机制（ASLR/DEP）？

### Q1

在9个程序中找到了7518个gadgets和5052个gadgets dispatcher，因此是普遍存在的

![image-20200526152840994](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-07-11-134404.jpg)



### Q2&Q3

通过对实际漏洞的攻击完成说明，见论文部分



## Discussion

DOP目前已经实现了对ASLR、DEP、TASR防御的突破，但也可能可以通过以下方法进行防御

- memory security：通过检测恶意内存损坏来防止出现内存错误。
- Data-Flow Integrity：类似于CFI在控制流完整性上的防御。
- Fine-grained Data-Plane Randomization：细粒度的数据面随机化可以缓解 DOP 攻击，因为 DOP 仍然需要获取某些非控制数据指针的地址。
- Hardware and Software Fault Isolation:内存隔离被广泛用于防止未经授权访问高权限资源，只有合法的代码区域才能访问特定的资源，这样可以防止一些直接的数据破坏攻击。

总体来说，上述保护措施都会对程序执行带来极大的开销，只是能用来防御，但也需要考量效率问题。


