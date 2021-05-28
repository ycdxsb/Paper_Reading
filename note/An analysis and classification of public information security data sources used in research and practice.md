## Abstract

是对当前公共信息安全数据源的分类与分析。对于信息安全数据来源多样且质量不一的情况进行研究分析，从六个维度进行分类和比较：(1) Type of information, (2) Integrability, (3) Timeliness, (4) Originality, (5) Type of Source,and (6) Trustworthiness。共收集和比较了68个公开的信息安全数据源，结果表明由于来源不同，数据异构繁多，加大了统一集成和使用的难度。

## Introduction

信息安全数据源：提供有关脆弱性、威胁、攻击、风险、受影响资产或可用对策的信息源

例如NVD、twitter的数据源等

当前研究gap：至今为止，对这些数据源的实证研究不多见，缺乏对这些数据源可用性、特征、依赖性和如何使用的系统且全面的概述。也没有对这些数据源的对比结果。

研究目标：对数据源进行分类，定性定量分析。

研究从以下三个问题入手：1、怎么分类；2、特征是什么；3、数据源之间有什么依赖关系



## Related work

- Steinberger et al. 分析现有用例，根据数据交换格式和协议，给出了结构化认识
- Hernandez-Ardieta et al. 提出基于交换格式的实时信息安全数据共享模型
- Rader and Wash 分析三类安全数据源：文章、网页、个人经历，发现主要内容为attack和结果
- Massacci and Nguyen 分析14个漏洞库，比较信息格式
- Tripathi and Singh 对几个漏洞库的漏洞分类方案进行分析，希望提出更高的分类方案
- Tounsi and Rais 对不同的威胁情报类型进行了分类。关注新的标准、趋势和技术问题。
- Mavroeidis and Bromander 对共享标准和策略进行分类
- Zhao and White概述了信息安全数据共享的重要性，并提供了重要共享的信息安全数据类型列表。
- .....

总结：目前的研究大多集中在信息安全数据交换或威胁情报共享方面，而对脆弱性数据库等信息安全数据源的分析研究还不多见。



## Research methodology

将问题2划分为很多子问题如下：

- 2.1 数据源存在哪些特征
- 2.2 信息结构是什么
- 2.3 获取数据的接口是什么
- 2.4 谁提供了这些信息
- 2.5 信息分享的时间
- 2.6 提供的信息是最原始的信息吗

问题3 划分：

- 3.1 提供的不同类型的信息之间有什么关系
- 3.2 接口如何与提供的不同类型的信息相关？

整体章节结构如下：

![image-20200415194603766](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-07-11-134610.jpg)



### Literature Review

基于snowballing方法，方法步骤如下：

- 定义文章的起始点集合
- 执行snowballing迭代（包括向前snowballing，例如确定引用被检查论文的新论文，以及向后snowballing，例如查看所考虑论文的参考文献）



定义起始点：通过关键字搜索各大数据库获得对应的文章（遵从snowballing的5大原则）

迭代：前向后向各进行三十次迭代直到没有新的paper进入集合，通过引用和被引等信息，经过blabla最后选出了42份优质论文



### Data collection on twitter

利用关键词，使用爬虫爬和CVE有关的tweets，pattern匹配`CVE-\d{4}-\d{4}`，（现在这个pattern已经不够用了）

一共搜集到了20160523-20180327间的709880个tweets，平均每个tweet中有0.8个url指向了包含详细信息的网页。一共有11437个不同的详细信息链接，选取了其中的top50



### Exploratory survey 

调研公共安全数据源的使用，通过问卷的方式对29个大公司进行调研：What public available information security data sources are you using as input to information security risk management processes?

让他们从87个备选数据源中挑选最常用的3个，然后最后根据调研选出了32个数据源



### Selection of information security data sources

从上面的42，50，32中选取开源、英语并且和attack、risk等相关的，其他的商用、非英语什么的丢掉



### Development of classification taxonomy

没用的章节



### Classification and analysis of information security data sources

好像也没什么用，总之是为了减小个人看法对分类结果的影响，分类也是人工分的



## Results

### Classification taxonomy

分类结果如下：

![image-20200415203957931](https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-07-11-134616.jpg)



根据信息类型 按照IEC2014划分为Vulnerability、Threat、Countermeasure、Attack、Risk、Asset

根据可集成性 按照IEC/ISO27005，描述了信息自动化聚集的程度，分为结构性的，非结构性的格式和接口等

根据及时性可以分为常规的日报月报和突发两种

根据独创性分为一手和二手资料

根据信息源类型分为 新闻网页、博客、安全产品网页、漏洞库、邮件、社交网络等

根据信赖程度分为 可信性、可追溯性、反馈机制



后面的东西与课程作业内容不是很相关所以就不看了，都是回答前面抛出的的questions

这六个分类维度和选题还算有点关系



