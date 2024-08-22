## 运行流程

**Job->Stage->Task开发完一个应用以后，把这个应用提交到Spark集群，这个应用叫Application。这个应用里面开发了很多代码，这些代码里面凡是遇到一个action操作，就会产生一个job任务。**

一个Application有一个或多个job任务。job任务被DAGScheduler划分为不同stage去执行，stage是一组Task任务。Task分别计算每个分区partition上的数据，Task数量=分区partition数量。

 

**Spark如何划分Stage**：会从执行action的最后一个RDD开始向前推，首先为最后一个RDD创建一个stage，向前遇到某个RDD是宽依赖，再划分一个stage。stage之间有shuffle过程

**一、 窄依赖与宽依赖**

针对不同的转换函数，RDD之间的依赖关系分为窄依赖（narrow dependency）和宽依赖（wide dependency，也叫shuffle dependency）。

**1.1 窄依赖**

窄依赖是指**1个父RDD分区对应1个子RDD的分区。**换句话说，一个父RDD的分区对应于一个子RDD的分区，或者多个父RDD的分区对应于一个子RDD的分区。所以窄依赖又可以分为两种情况：

- 1个子RDD的分区对应于1个父RDD的分区，比如map，filter，union等算子
- 1个子RDD的分区对应于N个父RDD的分区，比如co-partioned join



**1.2 宽依赖**

宽依赖是指**1个父RDD分区对应多个子RDD分区。**宽依赖有分为两种情况

- 1个父RDD对应非全部多个子RDD分区，比如groupByKey，reduceByKey，sortByKey
- 1个父RDD对应所有子RDD分区，比如未经协同划分的join
- 宽依赖之间会划分stage，而Stage之间就是Shuffle



**3. DAG**

**RDD之间的依赖关系就形成了DAG（有向无环图）**

**二、shuffle**

**1：shuffle write阶段**

主要就是在一个stage结束计算之后，为了下一个stage可以执行shuffle类的算子(比如reduceByKey，groupByKey)，而将每个task处理的数据按key进行“分区”。所谓“分区”，就是对相同的key执行hash算法，从而将相同key都写入同一个磁盘文件中，而每一个磁盘文件都只属于reduce端的stage的一个task。在将数据写入磁盘之前，会先将数据写入内存缓冲中，当内存缓冲填满之后，才会溢写到磁盘文件中去。

那么每个执行shuffle write的task，要为下一个stage创建多少个磁盘文件呢?很简单，下一个stage的task有多少个，当前stage的每个task就要创建多少份磁盘文件。比如下一个stage总共有100个task，那么当前stage的每个task都要创建100份磁盘文件。如果当前stage有50个task，总共有10个Executor，每个Executor执行5个Task，那么每个Executor上总共就要创建500个磁盘文件，所有Executor上会创建5000个磁盘文件。由此可见，未经优化的shuffle write操作所产生的磁盘文件的数量是极其惊人的。

**2：shuffle read阶段**

shuffle read，通常就是一个stage刚开始时要做的事情。此时该stage的每一个task就需要将上一个stage的计算结果中的所有相同key，从各个节点上通过网络都拉取到自己所在的节点上，然后进行key的聚合或连接等操作。由于shuffle write的过程中，task给Reduce端的stage的每个task都创建了一个磁盘文件，因此shuffle read的过程中，每个task只要从上游stage的所有task所在节点上，拉取属于自己的那一个磁盘文件即可。

shuffle read的拉取过程是一边拉取一边进行聚合的。每个shuffle read task都会有一个自己的buffer缓冲，每次都只能拉取与buffer缓冲相同大小的数据，然后通过内存中的一个Map进行聚合等操作。聚合完一批数据后，再拉取下一批数据，并放到buffer缓冲中进行聚合操作。以此类推，直到最后将所有数据到拉取完，并得到最终的结果。

## Shuffle

**一、shuffle简介**

Shuffle描述着数据从map task输出到reduce task输入的这段过程。shuffle是连接Map和Reduce之间的桥梁，Map的输出要用到Reduce中必须经过shuffle这个环节，shuffle的性能高低直接影响了整个程序的性能和吞吐量。因为在分布式情况下，reduce task需要跨节点去拉取其它节点上的map task结果。这一过程将会产生网络资源消耗和内存，磁盘IO的消耗。通常shuffle分为两部分：Map阶段的数据准备和Reduce阶段的数据拷贝处理。一般将在map端的Shuffle称之为Shuffle Write，在Reduce端的Shuffle称之为Shuffle Read.

**二、shuffle类型**

在Spark 1.2以前，默认的shuffle计算引擎是HashShuffleManager，因为HashShuffleManager会产生大量的磁盘小文件而性能低下，在Spark 1.2以后的版本中，默认的ShuffleManager改成了SortShuffleManager。SortShuffleManager相较于HashShuffleManager来说，有了一定的改进。主要就在于，每个Task在进行shuffle操作时，虽然也会产生较多的临时磁盘文件，但是最后会将所有的临时文件合并(merge)成一个磁盘文件，因此每个Task就只有一个磁盘文件。在下一个stage的shuffle read task拉取自己的数据时，只要根据索引读取每个磁盘文件中的部分数据即可。

**2.1 hashshuffle**

**2.2 sortshuffle**

**三、shuffle调优**

主要是调整缓冲的大小，拉取次数重试重试次数与等待时间，内存比例分配，是否进行排序操作等等

- spark.shuffle.file.buffer

参数说明：该参数用于设置shuffle write task的BufferedOutputStream的buffer缓冲大小（默认是32K）。将数据写到磁盘文件之前，会先写入buffer缓冲中，待缓冲写满之后，才会溢写到磁盘。

调优建议：如果作业可用的内存资源较为充足的话，可以适当增加这个参数的大小（比如64k），从而减少shuffle write过程中溢写磁盘文件的次数，也就可以减少磁盘IO次数，进而提升性能。在实践中发现，合理调节该参数，性能会有1%~5%的提升。

- spark.reducer.maxSizeInFlight：

参数说明：该参数用于设置shuffle read task的buffer缓冲大小，而这个buffer缓冲决定了每次能够拉取多少数据。

调优建议：如果作业可用的内存资源较为充足的话，可以适当增加这个参数的大小（比如96m），从而减少拉取数据的次数，也就可以减少网络传输的次数，进而提升性能。在实践中发现，合理调节该参数，性能会有1%~5%的提升。

- spark.shuffle.io.maxRetries：huffle read task从shuffle write task所在节点拉取属于自己的数据时，如果因为网络异常导致拉取失败，是会自动进行重试的。该参数就代表了可以重试的最大次数。（默认是3次）
- spark.shuffle.io.retryWait：该参数代表了每次重试拉取数据的等待间隔。（默认为5s）

调优建议：一般的调优都是将重试次数调高，不调整时间间隔。

- spark.shuffle.memoryFraction：

参数说明：该参数代表了Executor内存中，分配给shuffle read task进行聚合操作的内存比例。

- spark.shuffle.manager

参数说明：该参数用于设置shufflemanager的类型（默认为sort）。Spark1.5x以后有三个可选项：

Hash：spark1.x版本的默认值，HashShuffleManager

Sort：spark2.x版本的默认值，普通机制，当shuffle read task 的数量小于等于spark.shuffle.sort.bypassMergeThreshold参数，自动开启bypass 机制

tungsten-sort：

## 缓存

**缓存级别**

MEMORY_ONLY : 将 RDD 以反序列化 Java 对象的形式存储在 JVM 中。如果内存空间不够，部分数据分区将不再缓存，在每次需要用到这些数据时重新进行计算。这是默认的级别。

MEMORY_AND_DISK : 将 RDD 以反序列化 Java 对象的形式存储在 JVM 中。如果内存空间不够，将未缓存的数据分区存储到磁盘，在需要使用这些分区时从磁盘读取。

MEMORY_ONLY_SER : 将 RDD 以序列化的 Java 对象的形式进行存储（每个分区为一个 byte 数组）。这种方式会比反序列化对象的方式节省很多空间，尤其是在使用 fast serializer时会节省更多的空间，但是在读取时会增加 CPU 的计算负担。

MEMORY_AND_DISK_SER : 类似于 MEMORY_ONLY_SER ，但是溢出的分区会存储到磁盘，而不是在用到它们时重新计算。

DISK_ONLY : 只在磁盘上缓存 RDD。

MEMORY_ONLY_2，MEMORY_AND_DISK_2，等等 : 与上面的级别功能相同，只不过每个分区在集群中两个节点上建立副本。

OFF_HEAP（实验中）: 类似于 MEMORY_ONLY_SER ，但是将数据存储在 off-heap memory，这需要启动 off-heap 内存。

**cache，persist区别**

cache只有一个默认的缓存级别MEMORY_ONLY，即将数据持久化到内存中，而persist可以通过传递一个 StorageLevel 对象来设置缓存的存储级别。

**持久化的RDD释放**

Spark 自动监控各个节点上的缓存使用率，并以最近最少使用的方式（LRU）将旧数据块移除内存。如果想手动移除一个 RDD，而不是等待该 RDD 被 Spark 自动移除，可以使用 RDD.unpersist() 方法