


> 作者设计了一个可扩展的漏洞发现工具(Sys)，并且在已经被好多自动化工具检查过的软件中发现了一些漏洞，比如说Chrome，Firefox，以及sqlite3。
>
> 整个系统分为两个部分：首先通过静态分析定位可能存在漏洞的地方，然后在对这些备选项通过符号执行的方法进一步确认。这样就在漏洞发现的速度 和 准确率上得到了一个平衡。



### Introduction

研究原因：现在的大厂对产品安全问题都十分重视,Chrome、Firefox、Sqlite都是经过好多Fuzzer的检测才发布，所以想要在这些产品里找bug是难上加难，之前实验室里发布的符号执行工具KLEE已经不能解决这个问题了。

众所周知，静态分析：速度快但不准确，动态符号执行：速度慢准确。所以两者结合就在速度和准确度上做了一个平衡，这也是Sys的想法。

第一步：静态分析pass确定可能的error点。并且用户可以写自己的静态分析规则来定位potential error

第二步：对第一步的结果进行符号执行确认，去除误报。用户也可以写自己的符号习性checker。

Sys的符号执行并不是执行所有的片段，而是执行一部分代码，因为大部分的bug都是存在于一小部分上下文中的，比如说找无限循环，只要看for循环就可以了，这样也减少了符号执行的资源开销。

![image-20200827161021264](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-08-27-081021.png)

以上就是Sys在三个软件中找到的bugs总结。

因为在现实中的软件都是不一样且有高度交互的，所以在设计系统的时候为了用户定制，也实现了一套语言(DSL)，用户使用DSL语言在上层写自己的checker，然后剩下的转换都交给Sys完成。

**贡献点**：

- Sys系统和自带的5个checker
- 一种大规模软件符号执行的方法
- DSL语言，方便用户自定义



### System Overview

拿一个Sqlite的例子来说，用户只需要提供一个checker和一个LLVM的IR文件给Sys，就可以得到最终的bug报告。感觉还是很易用的呢，不过看了他论文实验的机器配置，虽然开源了代码，但让人丝毫没有想用的兴趣

![image-20200827161706760](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-08-27-081707.png)

Sys发现bug三部曲：

- 静态扫描
- 动态符号执行确认
- 总结报告

![image-20200827164121733](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-08-27-084122.png)

#### Static

静态扫描部分是允许用户扩展的，也就是上图里的extenson。这一步和其他的静态审计工具差不多，先使用LLVM生成中间IR文件，然后解析CFG，根据用Haskell语言写的extension检测bug。

但也和其他的静态工具有一点不同，之前的静态工具为了低误报都检查的比较细，但Sys不需要,因为它的误报低有动态符号执行来保证,所以在这一步可以选择检测出较多的可能bug,然后交给符号执行。

下面是一个Static 扩展的例子，用来找内存越界的问题

![image-20200827164843936](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-08-27-084844.png)

#### Symbolic

- 自动符号执行完整路径
- 使用用户定义的checker判断是否为漏洞

Sys的符号执行在内存拷贝的基础上进行，在IR上建立约束关系和逻辑表达式，然后加入用户的checker，最后使用SMT求解器求解关系是否可达。下面是一个Symbolic checker的例子

![image-20200827204415589](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-08-27-124415.png)

同时Sys在执行的时候也会跳过无关的语句和函数（类似于函数切片的思想，去除无关语句的执行，提高效率）



#### Unknown state

由于只执行部分上下文，所以中间一些相关变量的状态都是没法确定的，这也是Sys的一个不足之处。所以Sys在执行时需要自己去伪造这些状态，并且确保这些丢失的约束信息不会造成误报。



在第一点上，Sys采用的是懒分配的方法，也就是用的时候再分配内存够。比如说当出现解引用的时候，就分一块内存给变量。但由于位置的状态，所以造成误报是可能存在的，但是Sys有可以确保这种情况尽量的少：

1. Sys会探索所有可能的路径
2. 只要找部分上下文错误，而不是确保函数都正确
3. checker提供的漏洞存在时满足的条件信息
4. 大量的误报必然存在根本原因，Sys可以通过ad hoc,checker的定义技巧解决这个问题

### SysDSL design

算是一种好的语言设计的原则吧：

- 基于特定领域的，带有领域特性

- 有足够表达能力的，就是尽量保证能做自己想做的所有事

- 简单，比如python

- 多类型且安全的

  

#### Static extensiions

静态分析在LLVM的Bytecode上完成，理由如下：

1. 可以把静态pass和动态pass结合在一起
2. 在LLVM的IR上可以对任何语言进行扩展
3. 可以看到C++语言相对底层的东西
4. 可以考虑到编译的优化选项



#### SysDSL

使用SysDSL设计好符号执行的checker以后，会自动的将对应的LLVM指令转换成SMT约束表达，包括算数表达，比较，类型转换等等。

同时SysDSL也完成了对每一条LLVM指令的表达，这样可以让用户更好的实现checker的功能

![image-20200828152605972](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-08-28-072606.png)



### Memory Design

核心：将所有的对象什么使用的内存表示为连续的数组

原因：

- 现在的SMT求解器对数组的支持很好
- 对于C++中的多级指针，用数组可以很方便的表示，比如`***P`用数组表示为`mem[mem[mem[p]]]`

Sys也采用了Valdrind等工具一样的Shadow memory方法



缺点：

- 慢！
- 内存读取越界问题（可通过一些方法解决）



### Using Sys to find bugs

通过实例索命Sys的两个优点：

- 对漏洞的表达能力
- 找新bug的效率

我发现这个论文用了好大的篇幅来一遍一遍的吹，服了



#### uninitialized memory

静态checker：一个变量分配了内存，但没有明显的写入

动态checker：用shadow memory的看是不是真的没有对那块内存进行写入

![image-20200828155101240](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-08-28-075101.png)

然后也是在Chrome、Firefox、FreeBSD里找到了一些变量未初始化漏洞

![image-20200828155211996](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-08-28-075212.png)

#### Heap out-of-bounds

![image-20200828155353032](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-08-28-075353.png)

#### Concrete out-of-bounds

![image-20200828155439518](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-08-28-075439.png)

#### Unvalidated user data

> The checker traces untrusted values copied from user space, using the solver to flag errors if (1) an untrusted value used as an array index can be enormous; or (2) an untrusted value passed as a size parameter (e.g., to memcpy) could cause overflow.

### Conclusion

总的来说：Sys结合了静态分析和动态符号执行的优点，在速度和准确率上做了一个平衡，并且比较好的是，由于只符号执行部分上下文，所以在大型软件上有很大的优势，并且设计的SysDSL语言和Sys系统更配哦。

github官方出品的CodeQL工具，让我们可以静态的写checker查询，但没有符号执行，所以都得一个一个确认，如果能够自己在CodeQL的结果中写一套符号执行帮我们过滤到误报，感觉也是一个不错的选择



