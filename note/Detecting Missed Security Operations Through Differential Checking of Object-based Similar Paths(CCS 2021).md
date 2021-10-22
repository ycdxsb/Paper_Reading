---
title: >-
  Detecting Missed Security Operations Through Differential Checking of
  Object-based Similar Paths(CCS 2021)
tags:
  - paper
  - security
  - automatic analyse
author: ycdxsb
categories:
  - papers
  - security
  - automatic_analyse
abbrlink: e6c8cbe9
date: 2021-10-22 23:57:00
---
<!--toc-->


> 这是一篇卢康杰教授组在 CCS 2021被接收的论文，使用 基于对象的相似检查路径 差异检查 来检测 遗漏的安全操作

<!--more-->
## 研究介绍

**研究背景**

大型项目在实践的时候往往通过各种安全操作来确保项目的安全性，比如对一些变量的检查、lock/unlock资源、引用计数等等。

但是在大型项目中，有的函数代码动辄几千行，因此遗漏这种安全操作的情况十分常见，而且一旦遗漏这些安全操作，造成的影响也十分严重。通过分析发现，NVD中61%的漏洞都是由于安全操作的缺失造成的，而在Linux Kernel中这个比例达到了66%。

![image-20211019114137869](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2021-22-10-4LqUJ85BSoQebGk.png)

**研究现状**

对于这一类问题的研究，当前比较流行的做法是从项目中通过聚类等方法收集相似的代码片段，然后在同一个大类中通过投票的方法，找到检查不一致的代码认为可能存在漏洞。

这种方法不可避免的存在漏报，原因如下：

1. 原因1：很多代码片段可能是唯一的，因此并不能找到足够的相似代码片段。在投票时往往需要设置一个阈值，现有研究基本都采用0.8作为阈值，因此也就是说同一个大类中至少需要5个相似片段才能找到不一致的代码
2. 原因2：代码片段切片的粒度难以控制，为了使交叉检查具有足够的可扩展性以处理大型程序，现有方法通常抽象其代码表示或使用特定规则来限制切片生成，这样会导致损失一些有价值的代码片段
3. 原因3：少数服从多数的原则不是绝对的

**研究贡献**

- 提出了一个新的检测 安全操作缺失的工具 IPPO (Inconsistent Path Pairs as a bug Oracle)
- 提出了基于对象的相似路径对构建方法
- 在Linux Kernel ,Openssl, Freebsd Kernel,PHP项目中分别找到了154, 5, 1, 1个安全问题（82 refcount leak bugs, 57 memleak bugs, 10 missing check bugs, 7 use-after-free bugs, and 5 missing unlock bugs）



## 安全检查缺失漏洞

### 是什么

**示例1**： CVE-2019-8980 memory leak，当第9行检测通过时，第6行成功分配的buf未正确释放

![image-20211019114541092](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2021-22-10-TfQD197PiUk5mlq.png)

**示例2** ：CVE-2019-12819，第8行释放多余，因为出错时，外部xlr_setup_mdio Line 21也会对其进行释放

![image-20211019114946898](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2021-22-10-j82z61dTmcsMlb5.png)

### 什么影响

- 121 个Linux安全补丁中69个是安全操作缺失导致的(2019.1-8)
- 漏洞影响
  - Dos
  - 内存损坏
  - 信息泄露
  - 提权
  -  代码执行

### 为何出现

主要原因由两个：

1. 随着项目代码的增多，执行路径呈现指数型增长，因此开发者很容易遗漏安全检查操作
2. 不友好的API设计，导致即使有着完善的文档，开发者也会错误使用这些API

### 怎么检测

检测方法分为两类：

1. 根据已知漏洞去项目里定位/检索漏洞 => 只能找已知bug，不能找未知bug
2. 利用统计信息来推断遗漏安全操作的正确性 => 对于相似片段多还好，但是少了就不行

## IPPO系统概览

检测依据：如果两条路径对于相同的对象，存在相似的函数使用，那么两条路径对该对象的安全操作应该是相似的

![image-20211019114439981](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2021-22-10-tDxp4ZHB3rkRIlK.png)



- 第一步：预处理
  - 将源代码转换为LLVM IR
  - 生成全局函数调用图，控制流图，并对循环进行展开
- 第二步：程序分析

  - 安全操作检测
    - 安全检查
    - 资源的 分配/释放
    - 引用计数 增加/减少
    - 锁函数 lock/unlock
  - 安全对象提取
  - 基于对象的相似路径对收集
- 差异分析
  - 在路径对中检测安全操作缺失
  - 生成报告

## 基于对象的相似路径对构建技术

### 安全对象

首先需要的就是找安全对象，本文将安全操作中涉及的重要变量作为安全对象，目前专注于下面几类

- checker涉及的变量

```
if(var == null){
   return error;
}
```

- alloc/release涉及的变量

```
buf = alloc(var);
free(buf);
```

- refcount涉及的变量

```
refcount_set(var);
refcount_inc(var);
```

- lock/unlock涉及的变量

```
atomic(var);
mutex(var);
semaphore(var);
...
```

### OSPP设计原则

#### 代码表示

代码表示的粒度决定了错误检测的上限，本文选择控制流路径作为基本单元，同时对相似的代码段进行建模。 它包含相对更丰富的语义信息，使分析对路径敏感。 具体来说，控制流路径由控制流图 (CFG) 中的一系列连贯的基本块组成。

#### 敲黑板

IPPO中最重要的组成部分就是OSSP的构建

- 如果相似度分析过于宽松，可能会出现不相似的路径配对，导致误报

- 如果相似性分析过于严格，可能会错过有价值的相似路径对，导致漏报

因此，需要设计一种新的相似性分析方法，可以准确而广泛地识别相似路径对。对此，本文提出了提出了基于对象的相似路径对构建技术，提高了精度和覆盖率。

#### 构建OSPP的规则

为了构建OSPP，需要通过规则来判断是否是相似路径对，主要通过两个方面来判断：

- 安全对象的语义
- 安全对象的上下文

示例：

![image-20211019145752202](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2021-22-10-M57GsOzaAbt21Ef.png)

在上面的例子中，总共有4处错误处理路径，分别以10，17，24，32行结束，这些错误处理都十分类似

其中第17行和32行结束的路径free了chip变量，而第24行结束的路径并未正确free

从开发者的角度来看，上述4条路径对于chip对象来说是相似的，而安全人员需要判断它在返回前是否需要被释放

**规则1**

Rule 1: The two paths start at the same block and end at the same block in CFG. 

两条路径越接近，就越有可能以类似方式使用现有的一个对象。 因此，分析的目标是程序中最近的路径：共享相同开始和结束块的路径



在图5中，从第22行开始并在第24行和第32行返回的两个错误路径（虽然返回行号不同，但它们恰好在CFG中的返回块处结束）都释放了commpage_bak。 但是，如果我们考虑这两条路径具有不同的起始块（例如，一个从第 22 行开始，另一个从第 29 行开始），那么我们可能会丢失重要信息（例如，在第 27 行释放 commpage_bak）并得出错误的结论。

**规则2**

Rule 2: The object has the same state in two paths

如果对象的源在两条路径的内部/外部，我们认为这些路径中对象的状态是相同的



图5中chip的源（来自dev at line 4）在上述所有错误路径之外，满足规则2。 如果chip的分配（初始化）在第26行，那么第24行返回的错误路径不需要释放chip



**规则3**

Rule 3: The two paths have the same SO-influential operations. 

两条路径中存在具有相同SO影响的操作

为了确定两条路径对于安全对象和目标对象安全操作存在相似的语义，提出了SO影响操作这个概念。

![image-20211019151129735](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2021-22-10-E3juG2BTNYtzlbh.png)

- 安全检查主要用于在失败时终止执行流程，返回未经检查的值通常不构成错误。因此，我们期望在被检查对象之后还有其他计算任务（函数调用、算术运算、内存操作等）

- 资源分配/释放受资源传播操作的影响（例如，资源变量被传播到全局变量，并且有专门的回调函数来释放它）

- 引用计数和锁定/解锁受任何其他引用计数器调整或锁定状态调整的影响。 

因此，SO影响可以确定安全操作是否在路径中是必需的

**规则4**

Rule 4: The two paths have the same sets of pre- and post-conditions against the object

两条路径对对象具有相同的前置和后置条件集。假设相似执行路径应当拥有相似的前置条件和后置条件



路径的前置条件是它的分支条件（例如，变量 err 是路径从图 5 中第 13 行开始的前置条件），而后置条件是路径对返回值的影响



**分析挑战**

挑战主要是大型项目带来的，

1. 路径枚举的路径爆炸问题：分析 OSPP 要求我们首先成对收集路径，路径收集的一个直接想法是收集函数中从入口开始到出口结束的所有路径，这会导致路径爆炸问题
2. 路径匹配导致的爆炸问题：有些路径对可能只满足部分 OSPP 规则，但在枚举时不能简单地丢弃它们，因为它们可能与其他路径配对

**解决方案**

 为了解决上述挑战，本文提出了以下技术

- 观察到路径爆炸的主要原因是冗余的公共消息。 因此，本文以一种新的方式收集满足规则 1 的路径对，只收集除了起始块和结束块之外不共享任何公共基本块的路径，在本文中称为减少的相似路径（RSP） 。设计了一个两阶段的 RSP 收集算法来有效地收集 RSP
- 将一个函数的CFG划分为不同的部分，每个部分的路径共享相同的返回值，在本文中称为基于返回值的图(RVGs)。 从 RVG 收集的路径本质上满足规则 4 中的后置条件

然后在上述分析结果中，再使用其他的规则进行筛选

###  RVG生成

#### 错误边识别

错误边是基于返回值的图 (RVG) 生成的关键组件。本文主要考虑两种返回值作为后置条件：正常值和错误值，它们表示两种最常见的功能：正常功能和错误处理功能。如果路径返回错误代码或调用错误处理函数，则将其视为错误处理路径。

错误边应满足以下条件之一：（1）它连接到错误返回值或错误处理函数，或（2）它的所有后续边都连接到某些错误返回值或错误处理函数。 IPPO 使用从返回指令开始的反向数据流分析来查找错误返回值的源，然后使用从源开始的正向数据流分析来收集 CFG 的所有错误边。这些错误边被记录在一个全局集合（ErrEdgeSet）中。

![image-20211019170642678](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2021-22-10-bMpG2mfDhCzwPx8.png)

#### 基于返回值的RVG子图生成

由于只考虑两种返回值，IPPO 为每个函数生成两个 RVG：一个图包含所有错误处理路径，另一个包含所有正常路径，没有任何错误处理路径

<img src="https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2021-22-10-image-20211020163632717.png" alt="image-20211020163632717" style="zoom:50%;" />

![image-20211019171435810](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2021-22-10-7NMWTVeqmYnC4Pi.png)

#### 相似路径生成

已经有了两个子图后，需要在两个子图中收集相似路径

![image-20211019172212488](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2021-22-10-4jIULV1RKHG2giY.png)

<img src="https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2021-22-10-SFypqAY8b9gZCKG.png" alt="image-20211020163749079" style="zoom:50%;" />

<img src="https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2021-22-10-hKy7YSFRqUHmtgE.png" alt="image-20211020164221005" style="zoom:50%;" />

#### 根据OSPP规则筛选

![image-20211019172400342](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2021-22-10-Mgnh3GeRsC7EuB9.png)

![image-20211020165412047](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2021-22-10-NWvAhKibcg1OnHa.png)

## IPPO实现

基于LLVM 10k C++实现，通过多个pass进行分析

- PASS1 : 循环展开和全局调用图构建
- PASS2 : 找出包装函数
- PASS3：检测安全操作
- PASS4：OSPP分析

### 安全操作检测

**安全检查**：

采用Crix的结果：当其中一个分支处理失败而另一个继续正常执行时， 将该 if 语句视为安全检查

**引用计数**

从Linux Kernel中采集引用计数函数，用来识别安全操作

**资源分配/释放**

采用Hector的思想，将分配识别为返回指针类型值的函数调用，并将释放识别为它在 CFG 路径中的最后一次使用，这应该是一个未经检查的调用，为了提高准确率，还要求这些函数中必须包含release和free等关键字

**锁函数 lock/unlock**

收集函数调用的名称中带有 _lock 和 _unlock 关键字的函数，且要求unlock函数必须与lock函数共享相同的参数

### 路径分析

#### 返回值分类

IPPO 中的路径分析和安全检查检测都需要对函数返回值进行高精度分类。

- Linux 内核为非空函数提供三种类型的返回值：布尔值、整数值和指针值。 以前的工作主要考虑整数错误返回值，称为错误代码。 对于 Linux 内核，返回 0 通常表示成功，否则返回非零值

-  FreeBSD 内核和 PHP 与 Linux 内核相似

-  OpenSSL 库，函数在成功时返回 1

本文将错误代码的概念进一步扩展到布尔值和指针值，区分如下：

- 具有指针类型的函数预期在成功时返回非空指针，在失败时返回空
- 具有布尔类型的函数应在成功时返回 true，在失败时返回 false

#### RSP构建

为了加快分析流程，在路径分析阶段不涉及每个函数，在构建RSP时， 如果当前函数不存在任何安全操作，则放弃对该函数的分析

#### OSPP规则检查

首先收集和配对 RSP，然后在检查其余 OSPP 规则之前执行差异检查，以进一步加快我们的分析。 

仅当 RSP 的一个路径包含安全操作而另一个不包含时，IPPO 才按顺序检查其余的 OSPP 规则

### 差分分析

对于一对相似路径，一个潜在的安全漏洞需要 OSPP 的一个路径来包含安全操作，而另一个则不包含安全操作。 一旦在 OSPP 中发现遗漏的安全操作，IPPO 会生成详细的不一致信息以供进一步人工确认。

## 系统评估

- 实验平台：MacBook Pro laptop with 16GB RAM and an Intel(R) Core(TM) i7 CPU with six cores (i7-8850H, 2.60GHz).
- 实验对象：Linux Kernel、Freebsd 12 、PHP 8.0.8、Openssl 3.0.0-alpha6

### 结果概览

![image-20211019185733438](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2021-22-10-kdrfEFIiBRm3HqP.png)

共检测出754个结果

其中181个引用泄露 68 个内存泄露, 12个检查缺失, 7 个Double Free/UAF 漏洞, 以及7个未解锁漏洞

在这些结果中

- 对pm_runtime_get_sync的误用导致了上百个漏洞

- 由 API 误用引起的错误共116 个，其中 56 个（48.3%）包含超过 100 行的源代码，18 个（15.5%）包含超过 200 行。
- IPPO 捕获的最长的错误函数拥有 613 行源代码。这证明了 IPPO 能够检测复杂功能中的错误。实际上，上述长函数中有17个bug早在五年前就引入了，其中4个bug已经存在了十多年。

### 与交叉检查工具比较

APISan和Crix基于交叉检查

FICS 基于机器学习

![image-20211019192324914](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2021-22-10-kcxZjGs7WJHyeAa.png)

原因：

- FICS对大型项目的服务器要求太高，无法分析Linux Kernel
- Crix设计用于检查missing check漏洞，但由于IPPO分析结果对于Crix来说，缺乏足够的上下文信息，因此只检测出1个
- APISan 考虑构建语义的路径中涉及到的所有条件。 但是，正如OSPP规则3中提到的，许多中间条件判断和操作对安全操作的使用没有影响，这使得APISan的相似性分析的鲁棒性很差

### 与配对分析工具相比

HERO是最先进的配对分析工具，可以精确检测成对使用的函数以及由无序错误处理（错误处理的缺失、冗余和错误顺序）引起的错误。 HERO 涵盖的 bug 类型包括 refcount 泄漏、内存泄漏、use-after-free/double-free 和不正确的 lock/unlock，这与 IPPO 支持的 bug 类型非常相似

![image-20211019194009500](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2021-22-10-P8DGWVCBZ9aOkRn.png)

## 结语

总体的思路还是很不错的，误报高是因为整体分析中的很多步骤目前的方法都不够完美，不是IPPO本身的问题。

可以看到IPPO为了解决路径爆炸问题，将分析目标框定在单个函数中，所以分析的范围只是单个函数中存在的不一致问题，而不同函数间的相似代码不一致性并未解决，我猜这会是Kangjie Lu组后面会继续做的工作。
