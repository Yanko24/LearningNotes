#### Hadoop常见的面试题

##### 1. 请说下HDFS的读写流程？

###### HDFS读流程



###### HDFS写流程

##### 2. HDFS在读取文件的时候，如果其中一个块突然损坏了怎么办？

客户端读取完DataNode上的块之后会进行checksum验证，也就是把客户端读取到本地的块与HDFS上原始块进行校验，如果发现校验结果不一致，客户端就会通知NameNode，然后再从下一个拥有该block副本的DataNode继续读取。

##### 3. HDFS在上传文件的时候，如果其中一个DataNode突然挂掉了怎么办？

客户端上传文件时与DataNode建立pipeline管道，管道正向是客户端向DataNode发送的数据包，管道反向是DataNode向客户端发送ack确认，也就是正确接收到数据包之后发送一个确认收到的应答，当DataNode突然挂掉了，客户端接收不到这个DataNode发送的ack确认，客户端会通知NameNode，NameNode检查该块的副本与规定的不符，NameNode会通知DataNode去复制副本，并将挂掉的DataNode做下线处理，不再让它参与文件上传与下载。

##### 4. NameNode在启动时会做那些操作？

##### 5. Secondary NameNode了解吗？它的工作机制是怎样的？

##### 6. Secondary NameNode不能恢复NameNode的全部数据，那如何保证NameNode数据存储安全？

##### 7. 在NameNode HA中，为什么会出现脑裂问题？怎么解决脑裂？

##### 8. 小文件过多会有什么危害，如何避免？

Hadoop上大量HDFS元数据信息存储在NameNode内存中，因此过多的小文件必定会压垮NameNode的内存。每个元数据对象约占150byte，所以如果有1千万个小文件，每个文件占用一个block，则NameNode大约需要2G空间。如果存储1亿个文件，则NameNode需要20G空间。

显而易见的解决这个问题的方法就是合并小文件，可以选择在客户端上传时执行一定的策略先合并，或者是使用Hadoop的CombineFileInputFormat<K,V>实现小文件的合并。

##### 9. 请说下HDFS的组织架构？

##### 10. 请说下MR中的Map Task的工作机制？

##### 11. 请说下MR中的Reduce Task的工作机制？

##### 12. 请说下MR中shuffle阶段？

shuffle阶段分为四个步骤：依次为：分区、排序、规约、分组，其中前三个步骤在map阶段完成，最后一个步骤在reduce阶段完成。shuffle是MapReduce的核心，它分布在MapReduce的map和reduce阶段。一般把从Map产生输出开始到Reduce取得数据作为输入之前的过程称之为shuffle。

**Collect阶段：**将MapTask的结果输出到默认大小为100M的环形缓冲区，保存的是key/value，Partition分区信息等。

**Spill阶段：**当内存中的数据量达到一定阈值的时候，就会将数据写入本地磁盘，在将数据写入磁盘之前需要对数据进行一次排序操作，如果配置了全局的combiner，还会将有相同分区号和key的数据进行排序。

**Merge阶段：**把所有溢出的临时文件进行一次合并操作，以确保一个MapTask最终只产生一个中间数据文件。

**Copy阶段：**ReduceTask启动Fetcher线程到已经完成MapTask的节点上复制一份属于自己的数据，这些数据默认会保存在内存的缓冲区汇中，当内存的缓冲区达到一定阈值的时候，就会将数据写到磁盘上。

**Merge阶段：**在RecudeTask远程复制数据的同时，会在后台开启两个线程对内存到本地的数据文件进行合并操作。

**Sort阶段：**在对数据进行合并的同时，会进行排序操作，由于MapTask阶段已经对数据做了局部的排序，ReduceTask只需要保证Copy的数据的最终整体有序性即可。

shuffle中的缓冲区大小会影响到MapReduce程序的执行效率，原则上说，缓冲区越大，磁盘IO的次数越少，执行速度就越快。缓冲区的大小可以通过参数调整，参数：`mapreduce.task.io.sort.mb`，默认为100MB。

##### 13. shuffle阶段的数据压缩机制了解吗？

在shuffle阶段，可以看到数据通过大量的拷贝，从map阶段输出的数据，都要通过网络拷贝，发送到reduce阶段，这一过程中，涉及到大量的网络IO，如果数据能够进行压缩，那么数据的发送量就会少的多。

hadoop当中支持的压缩算法：gzip、bzip2、LZO、LZ4、Snappy，这几种压缩算法综合压缩和解压缩的速率，谷歌的Snappy是最优的，一般都选择Snappy压缩。

##### 14. 在写MR时，什么情况下可以使用规约？

规约（combiner）是不能够影响任务的运行结果的，局部汇总，适用于求和类，不适用于求平均值，如果reduce的输入参数类型和输出参数类型是一样的，则规约的类可以使用reduce类，只需要在驱动类中指明规约的类即可。

##### 15. yarn集群的架构和工作原理知道多少？

##### 16. yarn的任务提交流程是怎样的？

##### 17. yarn的资源调度三种模型了解吗？

在Yarn中有三种调度器可以选择：FIFO Scheduler，Capacity Scheduler，Fair Scheduler。apache版本的hadoop默认使用的是Capacity Scheduler调度方式。CDH版本的默认使用的是Fair Scheduler调度方式。

**FIFO Scheduler**（先来先服务）：FIFO Scheduler把应用按提交的顺序排成一个队列，这是一个先进先出队列，在进行资源分配的时候，先给队列中最头上的应用分配资源，待最上头的应用需求满足后再给下一个分配，以此类推。FIFO Scheduler是最简单也是最容易理解的调度器，也不需要任何配置，但它并不适用于共享集群。大的应用可能会占用所有集群资源，这就导致其他应用被阻塞，比如有个大的任务在执行，占用了全部的资源，再提交一个小任务，则此小任务会一直被阻塞。

**Capacity Scheduler**（能力调度器）：对于Capacity Scheduler调度器，有一个专门的队列用来运行小任务，但是为小任务专门设置一个队列会预先占用一定的集群资源，这就导致大任务的执行时间会落后于使用FIFO Scheduler调度器的时间。

**Fair Scheduler**（公平调度器）：在Fair Scheduler调度器中，我们不需要预先占用一定的系统资源，Fair调度器会为所有运行的job动态的调整系统资源。比如：当第一个大的job提交时，只有这一个job在运行，此时它获得了所有集群资源，当第二个小任务提交后，Fair调度器会分配一半资源给这个小任务，让两个任务公平的共享集群资源。需要注意的是，在Fair调度器中，从第二个任务提交到获取资源会有一定的延迟，因为它需要等待第一个任务释放占用的Container。小任务执行完成之后也会释放自己占用的资源，大任务又获得了全部的系统资源。最终的效果就是Fair调度器即得到了高的资源利用率又能保证小任务及时完成。