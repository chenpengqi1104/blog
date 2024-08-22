**一.为什么使用HBase存储**

HBase(Hadoop Database) 是一个靠可靠性,高性能,可伸缩,面向列的分布式数据库

HBase与Hadoop的关系非常紧密,Hadoop的hdfs提供了高可靠性的底层存储支持,Hadoop MapReduce 为 HBase提供了高性能的计算能力,zookeeper为Hbase提供了稳定性及failover机制的保障. 同时其他周边产品诸如Hive可以与HBase相结合使在HBase进行数据统计处理变得简单,Sqoop为HBase提供了方便的RDBMS数据导入功能,使传统数据库的数据向HBase中迁移变得容易,spark等高性能的内存分布式计算引擎也可能帮助我们更加快速的对HBase中的数据进行处理分析

**二.Rowkey设计原则**

**1.长度原则**

Rowkey是一个二进制码流,可以是任意字符串,最大长度为64kb,实际应用中一般为10-100byte,以byte[]形式保存,一般设计成定长.建议越短越好,不要超过16个字节,原因如下:

数据的持久化文件HFile中时按照key-value存储的,如果Rowkey过长,例如超过100byte,那么1000w行的记录,仅Rowkey就需占用近1G空间.这样会极大影响HFile的存储效率

MemStore会缓存部分数据到内存中,若Rowkey字段过长,内存的有效利用率就会降低,就不能缓存更多的数据,从而降低检索效率

目前操作系统都是64位系统,内存8字节对齐,控制在16字节,8字节的整数倍利用了操作系统的最佳特性

**2.唯一原则**

必须在设计上保证Rowkey的唯一性.由于在HBase中数据存储是key-value形式,若向HBase中同一张表插入相同Rowkey的数据,则原先存在的数据会被新的数据覆盖

**3.排序原则**

HBase的Rowkey是按照ASCII有序排序的,因此我们在设计Rowkey的时候要充分利用这点

**4.散列原则**

设计的Rowkey应均匀的分布在各个HBase节点上

**三.Hbase的优化**

**1.表设计**

1)建表时就分区(预分区),rowkey设置定长(64字节),CF2到3个

2)Max Versio, Time to live, Compact&split

**2.写表**

1)多Htable并发写,提高吞吐量

2)Htable参数设置,手动flush,降低IO

3)WriteBuffer

4)批量写,减少网络I/O开销

5)多线程并发写,结合定时flush和写buffer(writeBufferSize),可以既保证在数据量小的时候,数据可以在较短时间内被flush(如1秒内),同时有保证在数据量大的时候,写buffer一满就及时进行flush

**3.读表**

1)多Htable并发读,提高吞吐量

2)Htable参数设置

3)批量读

4)释放资源

5)缓存查询结果