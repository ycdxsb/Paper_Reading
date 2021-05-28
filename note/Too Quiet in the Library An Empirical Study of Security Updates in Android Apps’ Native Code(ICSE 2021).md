


论文：https://arxiv.org/pdf/1911.09716.pdf

代码：https://github.com/salmanee/Librarian



### Abstract

**研究背景**：为了加速执行和复用，native code(.so)在安卓的app里十分常见，但是因为未及时更新安全补丁甚至直接忽略，导致可以长时间被利用

为了深入了解这类现象，作者选择2013年9月到2020年5月google play上最受欢迎的200个app进行调查

**研究重点**：对.so库和版本的识别，提出了LibRARIAN（LibRAry veRsion IdentificAtioN）方法，根据bin2sim准确识别库和版本

**研究结果**：

1. 具有已知CVE的易受攻击版本的53/200个流行应用（占26.5％），其中14个仍然易受攻击
2. 应用程序开发人员平均花费528.71±40.20天来更新安全补丁，而库开发人员在54.59±8.12天后发布安全补丁，是更新速度的10倍



**研究贡献**：

1. 定位.so库和版本的方法LibRARIAN，测试集上表现准确率为91.15%
2. 一个大型的数据库：7年内的200个app的7678个版本，包含66,684个native库



### LibRARIAN

![image-20210317144110955](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2021-19-03-image-20210317144110955.png)



#### 特征选择

![image-20210317144803901](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2021-19-03-image-20210317144803901.png)

MetaData:

- 库的导出变量
- 库的导入变量
- 库的导出函数
- 库的导入函数
- 库的依赖，例如a.so需要依赖b.so

Data：

- 代表版本的字符串常量，LibFoo-1.0.2



没有选择数据流和控制流，因为和编译相关，并不稳定，而且相关算法基本和代码大小呈现指数级复杂度

#### 构建Ground Truth

通过正则去匹配版本，构建Ground Truth，实现上是通过Angr的ELF模块进行解析，在只读串中进行搜索版本字符串，实现了对x86-64, x86, ARM, and ARM64架构下库的分析

也做了一点数据清洗，比如尽可能多的搜集不同架构，去除只在一种架构下出现的字符串

![image-20210317145345387](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2021-19-03-image-20210317145345387.png)



#### 相似度计算

比较某app中.so和Ground Truth中.so的相似度

![image-20210317160053731](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2021-19-03-image-20210317160053731.png)

最后设置一个最低的相似度阈值0.85

一些影响因素：

- 开发者的修改，例如删除某些函数，会导致相似度降低
- 为了减小大小，会去除无用模块，并且strip信息
- 一些架构下，导出的函数会更多



#### 极端情况处理

比如导出导入函数很少甚至没有的情况，以及比较结果很差的情况，会再次利用版本字符串去帮助确定版本

找版本字符串：对常见的版本字符串进行聚类，提取他们的组成正则表达式



### Evalutation

**回答三个问题**

1. LibRARIAN准确率和效率
2. Native library问题在安卓应用中多么普遍
3. 开发者对补丁的响应时间



**数据准备**：从google play搜集了200个软件，共7,678个不同历史版本

**发现1**：流行的Android app中，大约每个app拥有11个.so文件也就是native代码库，而某些app拥有超多数量的代码库，例如Instgram在其2013年12月到2020年的所有版本中一共包含了6677个不同版本的.so文件，最多的时候特定版本的Instgram居然包含了141个.so文件

**发现2**：提取特征2117个，最多的LibWave和Libtensorflow可以有79581个特征



**回答问题1：**

效率：

1. 提取特征很快，最复杂的LibTensorflow的分析只需要4分钟38秒
2. 分析很快，最慢224秒，最快97秒，平均118秒

准确率：

LibRARIAN 对46个不同库的904个样本的分析结果中有824 (91.15%) 个是正确的，即使是那些不正确的，推断出的版本号和正确版本的差别也很小



特征对准确率影响：

![image-20210317164045365](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2021-19-03-image-20210317164045365.png)



**回答问题2：**

![image-20210317164218050](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2021-19-03-image-20210317164218050.png)

1. 具有已知CVE的易受攻击版本的53/200个流行应用（占26.5％），其中14个仍然易受攻击，且这些app安装量惊人，攻击面很大

![image-20210317164418341](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2021-19-03-image-20210317164418341.png)



**回答问题3：**

![image-20210317164506377](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2021-19-03-image-20210317164506377.png)

1. 应用程序开发人员平均花费528.71±40.20天来更新安全补丁，而库开发人员在54.59±8.12天后发布安全补丁，是更新速度的10倍

![image-20210317164541238](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2021-19-03-image-20210317164541238.png)

![image-20210317164615253](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2021-19-03-image-20210317164615253.png)



### 总结

内容不难，但是论文中的分析写的好
