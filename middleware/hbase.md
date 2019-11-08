HBase是Apache Hadoop的数据库，能够对大型数据提供随机、实时的读写访问。HBase的目标是存储并处理大型的数据。HBase是一个开源的，分布式的，多版本的，面向列的存储模型。它存储的是松散型数据。

## 一、HBase架构

**HBase特性：**

1 高可靠性

2 高效性

3 面向列

4 可伸缩

5 可在廉价PC Server搭建大规模结构化存储集群

**HBase是Google BigTable的开源实现**，其相互对应如下：

　　　　　　　　　　**Google 　　　　　　　　　　 HBase**
文件存储系统 　　　  GFS 　　　　　　　　　　　  HDFS
海量数据处理 　　　  MapReduce Hadoop 　　　　MapReduce
协同服务管理　　　　Chubby 　　　　　　　　　　Zookeeper

 

**HBase关系图：**

![img](../assets/2012060400463595.jpg)

HBase位于结构化存储层，围绕HBase，各部件对HBase的支持情况：
**Hadoop部件　　　　　　　　　　　　作用**
HDFS　　　　　　　　　　　　　　高可靠的底层存储支持
MapReduce 　　　　　　　　　　   高性能的计算能力
Zookeeper 　　　　　　　　　　　  稳定服务和failover机制
Pig&Hive　　　　　　　　　　　　  高层语言支持，便于数据统计
Sqoop　　　　　　　　　　　　　  提供RDBMS数据导入，便于传统数据库向HBase迁移

**访问HBase的接口**

**方式　　　　　　　　　　　　特点　　　　　　　　　　　　　　场合**
Native Java API　　　　　　最常规和高效 　　　　　　　　　　 Hadoop MapReduce Job并行处理HBase表数据
HBase Shell　　　　　　　  最简单接口 　　　　　　　　　　　 HBase管理使用
Thrift Gateway　　　　　　利用Thrift序列化支持多种语言 　　  异构系统在线访问HBase表数据
Rest Gateway　　　　　　 解除语言限制 　　　　　　　　　　 Rest风格Http API访问
Pig　　　　　　　　　　　　Pig Latin六十编程语言处理数据 　　数据统计
Hive　　　　　　　　　　　 简单，SqlLike

**HBase 数据模型**

![img](../assets/2012060400513336.jpg)

组成部件说明：

Row Key：　　 　　Table主键 行键 Table中记录按照Row Key排序
Timestamp：  　每次对数据操作对应的时间戳，也即数据的version number
Column Family： 　列簇，一个table在水平方向有一个或者多个列簇，列簇可由任意多个Column组成，列簇支持动态扩展，无须预定义数量及类型，二进制存储，用户需自行进行类型转换

**Table&Region**

![img](../assets/2012060400532513.jpg)

1. Table随着记录增多不断变大，会自动分裂成多份Splits，成为Regions
2. 一个region由[startkey，endkey)表示
3. 不同region会被Master分配给相应的RegionServer进行管理

**两张特殊表：-ROOT- & .META.**

![img](../assets/2012060400544147.jpg)

.META. 　　记录用户表的Region信息，同时，.META.也可以有多个region
-ROOT- 　 记录.META.表的Region信息，但是，-ROOT-只有一个region
Zookeeper中记录了-ROOT-表的location
客户端访问数据的流程：
Client -> Zookeeper -> -ROOT- -> .META. -> 用户数据表
多次网络操作，不过client端有cache缓存

**HBase 系统架构图**

![img](../assets/2012060400561367.jpg)

组成部件说明
**Client：**
使用HBase RPC机制与HMaster和HRegionServer进行通信
Client与HMaster进行通信进行管理类操作
Client与HRegionServer进行数据读写类操作

**Zookeeper：**
Zookeeper Quorum存储-ROOT-表地址、HMaster地址
<u>HRegionServer把自己以Ephedral方式注册到Zookeeper中，</u><u>HMaster随时感知各个HRegionServer的健康状况</u>
Zookeeper避免HMaster单点问题

**HMaster：**
<u>HMaster没有单点问题，HBase中可以启动多个HMaster，通过Zookeeper的Master Election机制保证总有一个Master在运行</u>
主要负责Table和Region的管理工作：
1 管理用户对表的增删改查操作
2 管理HRegionServer的负载均衡，调整Region分布
3 Region Split后，负责新Region的分布
4 在HRegionServer停机后，负责失效HRegionServer上Region迁移

**HRegionServer：**
HBase中最核心的模块，主要负责响应用户I/O请求，向HDFS文件系统中读写数据

![img](../assets/2012060400572691.jpg)

HRegionServer管理一些列HRegion对象；
每个HRegion对应Table中一个Region，HRegion由多个HStore组成；
<u>每个HStore对应Table中一个Column Family的存储</u>；
Column Family就是一个集中的存储单元，故将具有相同IO特性的Column放在一个Column Family会更高效

**HStore：**
HBase存储的核心。由MemStore和StoreFile组成。
MemStore是Sorted Memory Buffer。用户写入数据的流程：

![img](../assets/2012060400574793.gif)

<u>Client写入 -> 存入MemStore，一直到MemStore满 -> Flush成一个StoreFile</u>，直至增长到一定阈值 -> 出发Compact合并操作 -> 多个StoreFile合并成一个StoreFile，同时进行版本合并和数据删除 -> 当StoreFiles Compact后，逐步形成越来越大的StoreFile -> 单个StoreFile大小超过一定阈值后，触发Split操作，把当前Region Split成2个Region，Region会下线，新Split出的2个孩子Region会被HMaster分配到相应的HRegionServer上，使得原先1个Region的压力得以分流到2个Region上
由此过程可知，HBase只是增加数据，有所得更新和删除操作，都是在Compact阶段做的，所以，用户写操作只需要进入到内存即可立即返回，从而保证I/O高性能。

**HLog**
引入HLog原因：
在分布式系统环境中，无法避免系统出错或者宕机，<u>一旦HRegionServer以外退出，MemStore中的内存数据就会丢失，引入HLog就是防止这种情况</u>
工作机制：
<u>每个HRegionServer中都会有一个HLog对象</u>，HLog是一个实现Write Ahead Log的类，每次用户操作写入Memstore的同时，也会写一份数据到HLog文件，HLog文件定期会滚动出新，并删除旧的文件(已持久化到StoreFile中的数据)。当HRegionServer意外终止后，HMaster会通过Zookeeper感知，HMaster首先处理遗留的HLog文件，将不同region的log数据拆分，分别放到相应region目录下，然后再将失效的region重新分配，领取到这些region的HRegionServer在Load Region的过程中，会发现有历史HLog需要处理，因此会Replay HLog中的数据到MemStore中，然后flush到StoreFiles，完成数据恢复。

**HBase存储格式**
HBase中的所有数据文件都存储在Hadoop HDFS文件系统上，格式主要有两种：
1 HFile HBase中KeyValue数据的存储格式，HFile是Hadoop的二进制格式文件，实际上StoreFile就是对HFile做了轻量级包装，即StoreFile底层就是HFile
2 HLog File，HBase中WAL（Write Ahead Log） 的存储格式，物理上是Hadoop的Sequence File

**HFile**

![img](../assets/2012060400581450.jpg)

图片解释：
HFile文件不定长，长度固定的块只有两个：Trailer和FileInfo
Trailer中指针指向其他数据块的起始点
File Info中记录了文件的一些Meta信息，例如：AVG_KEY_LEN, AVG_VALUE_LEN, LAST_KEY, COMPARATOR, MAX_SEQ_ID_KEY等
Data Index和Meta Index块记录了每个Data块和Meta块的起始点
Data Block是HBase I/O的基本单元，为了提高效率，HRegionServer中有基于LRU的Block Cache机制
每个Data块的大小可以在创建一个Table的时候通过参数指定，大号的Block有利于顺序Scan，小号Block利于随机查询
每个Data块除了开头的Magic以外就是一个个KeyValue对拼接而成, Magic内容就是一些随机数字，目的是防止数据损坏

HFile里面的每个KeyValue对就是一个简单的byte数组。这个byte数组里面包含了很多项，并且有固定的结构。

![img](../assets/2012060400583891.jpg)


KeyLength和ValueLength：两个固定的长度，分别代表Key和Value的长度
Key部分：Row Length是固定长度的数值，表示RowKey的长度，Row 就是RowKey
Column Family Length是固定长度的数值，表示Family的长度
接着就是Column Family，再接着是Qualifier，然后是两个固定长度的数值，表示Time Stamp和Key Type（Put/Delete）
Value部分没有这么复杂的结构，就是纯粹的二进制数据

**HLog File**

![img](../assets/2012060400590244.jpg)


HLog文件就是一个普通的Hadoop Sequence File，Sequence File 的Key是HLogKey对象，HLogKey中记录了写入数据的归属信息，除了table和region名字外，同时还包括 sequence number和timestamp，timestamp是“写入时间”，sequence number的起始值为0，或者是最近一次存入文件系统中sequence number。
HLog Sequece File的Value是HBase的KeyValue对象，即对应HFile中的KeyValue

## 二、存储模型

有了架构层面的保证，接下来的事情就只是关注于数据的具体存储了。这里就是每个Region所承担的工作了。

- 我们知道一个Region代表的是一张 Hbase表中特定Rowkey范围内的数据，

- 而Hbase是面向列存储的数据库，所以在一个Region中，有多个文件来存储这些列。

- Hbase中

  数据 列是由列簇来组织

  的，所以

  每一个列簇都会有对应的一个数据结构

  ，

  -  Hbase将列簇的存储数据结构抽象为Store，一个Store代表一个列簇。

![img](../assets/41a6ad6f-7077-3a4e-9257-46bc7948c928.png)

所以在这里也可以看出为什么在我们查询的时候要尽量减少不需要的列，

而经常一起查询的列要组织到一个列簇里：因为要需要查询的列簇越多，意味着要扫描越多的Store文件，这就需要越多的时间。

我们来深入Store中存储数据的方式。

Hbase的实现是用了一种LSM 树的结构！

LSM树是由B+树改进

首先来简单的看看B+树：

![img](../assets/134069a3-720c-3421-8bb0-659aa8cab9df.png)

这是一颗简单的B+树，含义不言而喻，这里不多分析，但是这种数据结构并不适合Hbase中的应用场景。

这样的数据结构在内存中效率是很高的，但是 Hbase中数据是存储在文件中的，如果按照这样的结构来存储，意味着我们每一次插入数据都要由一级索引找到文件再在文件中间作操作来保证数据的有序性， 这无疑是效率低下的。

所以Hbase采用的是LSM树的结构，这种结构的关键是，

- 每一次的插入操作都会先进入MemStore（内存缓冲区），
- 当 MemStore达到上限的时候，Hbase会将内存中的数据输出为有序的StoreFile文件数据（根据Rowkey、版本、列名排序，这里已经和列 簇无关了因为Store里都属于同一个列簇）。
- 这样会在Store中形成很多个小的StoreFile，当这些小的File数量达到一个阀值的时 候，Hbase会用一个线程来把这些小File合并成一个大的File。

这样，Hbase就把效率低下的文件中的插入、移动操作转变成了单纯的文件输出、 合并操作。

 由上可知，在Hbase底层的Store数据结构中，

- 每个StoreFile内的数据是有序的，
- 但是StoreFile之间不一定是有序的，
- Store只 需要管理StoreFile的索引就可以了。

这里也可以看出为什么指定版本和Rowkey可以加强查询的效率，因为指定版本和Rowkey的查询可以利用 StoreFile的索引跳过一些肯定不包含目标数据的数据。

 

![img](../assets/acabc47a-6acc-3630-a5d4-0e1538941177.png)