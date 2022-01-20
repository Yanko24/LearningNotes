#### Hadoop常见的面试题

##### 1. 请说下HDFS的读写流程？

**HDFS读流程：**

- client向namenode发送RPC请求，请求文件block的位置
- namenode收到请求之后会检查用户权限以及是否有这个文件，如果都符合，则会视情况返回部分或全部的block列表，对于每个block，NameNode都会返回含有该block副本的DataNode地址；这些返回的DataNode地址，会按照集群拓扑结构得出DataNode与客户端的距离，然后进行排序，排序两个规则：网络拓扑结构中距离client近的排靠前；心跳机制中超时汇报的DataNode状态为STALE，这样的排靠后
- client选取排序靠前的DataNode来读取block，如果客户端本身就是DataNode，那么将从本地直接获取数据（短路读取特性）
- 底层上本质是建立Socket Stream（FSDateInputStream），重复的调用父类DataInputStream的read方法，直到这个块上的数据读取完毕
- 当读完列表的block后，若文件读取还没有结束，客户端会继续向NameNode获取下一批的block列表
- 读取完一个block都会进行checksum验证，如果读取DataNode时出现错误，客户端会通知NameNode，然后再从下一个拥有该block副本的DataNode继续读
- read方法是并行的读取block信息，不是一块一块的读取；NameNode只是返回client请求包含块的DataNode地址，并不是返回请求块的数据
- 最终读取来的所有的block会合并成一个完整的最终文件

**HDFS写流程：**

- client客户端发送上传请求，通过RPC和namenode建立通信，namenode检查该用户是否有上传权限，以及上传的文件是否在HDFS对应的目录下重名，如果这两者有任意一个不满足，则直接报错，如果两者都满足，则返回给客户端一个可以上传的信息

- client根据文件的大小进行切分，默认128M一块，切分完成之后给namenode发送请求第一个block块上传到那些服务器上

- namenode收到请求之后，根据网络拓扑和机架感知以及副本机制进行文件分配，返回可用的DataNode的地址

  > Hadoop在设计时考虑到数据的安全与高效，数据文件默认在HDFS上存放三份，存储策略为本地一份，同机架内其他某一个节点上一份，不同机架的某一个节点上一份。

- 客户端收到地址之后与服务器地址列表中的一个节点如A进行通信，本质上就是RPC调用，建立pipeline，A收到请求之后会继续调用B，B再调用C，将整个pipeline建立完成，逐级返回client

- client开始向A上发送第一个block（先从磁盘读取数据然后放到本地内存缓存），以packet（数据包，64KB）为单位，A收到一个packet就会发送给B，然后B发送给C，A每传完一个packet就会放入一个应答队列等待应答

- 数据被分割成一个个的packet数据包在pipeline上依次传输，在pipeline反向传输中，逐个发送ack（命令正确应答），最终由pipeline中第一个DataNode节点A将pipelineack发送给client

- 当一个block传输完成之后，client再次请求NameNode上传第二个block，namenode重新选择三台DataNode给client

##### 2. HDFS在读取文件的时候，如果其中一个块突然损坏了怎么办？

客户端读取完DataNode上的块之后会进行checksum验证，也就是把客户端读取到本地的块与HDFS上原始块进行校验，如果发现校验结果不一致，客户端就会通知NameNode，然后再从下一个拥有该block副本的DataNode继续读取。

##### 3. HDFS在上传文件的时候，如果其中一个DataNode突然挂掉了怎么办？

客户端上传文件时与DataNode建立pipeline管道，管道正向是客户端向DataNode发送的数据包，管道反向是DataNode向客户端发送ack确认，也就是正确接收到数据包之后发送一个确认收到的应答，当DataNode突然挂掉了，客户端接收不到这个DataNode发送的ack确认，客户端会通知NameNode，NameNode检查该块的副本与规定的不符，NameNode会通知DataNode去复制副本，并将挂掉的DataNode做下线处理，不再让它参与文件上传与下载。

##### 4. NameNode在启动时会做那些操作？

NameNode数据存储在内存和本地磁盘，本地磁盘数据存储在fsimage镜像文件和edits编辑日志文件。

- 首次启动NameNode
  - 格式化文件系统，为了生成fsimage镜像文件
  - 启动NameNode
    - 读取fsimage文件，将文件内存加载进内存
    - 等待DataNode注册与发送block report
  - 启动DataNode
    - 向NameNode注册
    - 发送block report
    - 检查fsimage中记录的块数量和block report中的块的总数是否相同
  - 对文件系统进行操作（创建目录，上传文件，删除文件等）
    - 此时内存中已经有文件系统改变的信息，但是磁盘中没有文件系统改变的信息，此时会将这些改变信息写入edits文件中，edits文件中存储的是文件系统元数据改变的信息
- 第二次启动NameNode
  - 读取fsimage和edits文件
  - 将fsimage和edits文件合并成新的fsimage文件
  - 创建新的edits文件，内容为空
  - 启动DataNode

##### 5. Secondary NameNode了解吗？它的工作机制是怎样的？

Secondary NameNode是合并NameNode和edit logs到fsimage文件中，它的具体工作机制：

- Secondary NameNode询问NameNode是否需要checkpoint。直接带回NameNode是否检查结果
- Secondary NameNode请求执行checkpoint
- NameNode滚动正在写的edits日志
- 将滚动前的编辑日志和镜像文件拷贝到Secondary NameNode
- Secondary NameNode加载编辑日志和镜像文件到内存，并合并
- 生成新的镜像文件fsimage.chkpoint
- 拷贝fsimage.chkpoint到NameNode
- NameNode将fsimage.chkpoint重命名成fsimage

所以如果NameNode中的元数据丢失，是可以从Secondary NameNode恢复一部分元数据信息的，但不是全部，因为NameNode正在写的edits日志还没有拷贝到Secondary NameNode，这部分恢复不了。

##### 6. Secondary NameNode不能恢复NameNode的全部数据，那如何保证NameNode数据存储安全？

这个问题就要说到NameNode的高可用了，即NameNode HA，一个NameNode有单点故障的问题，那就配置双NameNode，配置有两个关键点，一是必须要保证这两个NN的元数据信息必须要同步的，二是一个NN挂掉之后另一个要立马补上。

- 元数据信息同步在HA方案中采用的是“共享存储”。每次写文件时，需要将日志同步写入共享存储，这个步骤成功才能认定写文件成功。然后备份节点周期从共享存储同步日志，以便进行主备切换。
- 监控NN状态采用zookeeper，两个NN节点的状态存放在ZK中，另外两个NN节点分别有一个进程监控程序，实时读取ZK中的NN状态，来判断当前的NN是不是已经宕机。如果standby的NN节点的ZKFC发现主节点已经挂掉，那么就会强制给原本的active NN节点发送强制关闭请求，之后将备用的NN设置为active。

##### 7. 在NameNode HA中，共享存储是怎么实现的知道吗？

NameNode共享存储方案有很多，比如Linux HA，VMware FT，QJM等，目前社区已经把由Clouderea公司实现的基于QJM（Quorum Journal Manager）的方案合并到HDFS的trunk之中并且作为默认的共享存储实现。

基于QJM的共享存储系统主要用于保存EditLog，并不保存FSImage文件。FSImage文件还是在NameNode的本地磁盘上。QJM共享存储的基本思想是来自于Paxos算法，采用多个称为JournalNode的节点组成的JournalNode集群来存储EditLog。每个JorunalNode保存同样的EditLog副本。每次NameNode写EditLog的时候，除了向本地磁盘写入EditLog之外，也会并行地向JournalNode集群之中的每一个JournalNode发送写请求，只要大多数（majority）的JournalNode节点返回成功就认为向JournalNode集群写入EditLog成功。如果有2N+1台JournalNode，那么根据大数据的原则，最多可以容忍N台JournalNode节点挂掉。

##### 8. 在NameNode HA中，为什么会出现脑裂问题？怎么解决脑裂？

> 假设NameNode1当前为Active状态，NameNode2当前为standby状态。如果某一时刻NameNode1对应的ZFFailoverController进程发生了“假死”现象，那么Zookeeper服务端会认为NameNode1挂掉了，根据前面的主备切换逻辑，NameNode2会替代NameNode1进入Active状态。但是此时NameNode1可能扔处于Active状态正常运行，这样NameNode1和NameNode2都处于Active状态，都可以对外提供服务。这种情况称为脑裂。

脑裂对于NameNode这类对数据一致性要求非常高的系统来说是灾难性的，数据会发生错乱且无法恢复。Zookeeper社区对这种问题的解决方法叫做fencing，中文翻译为隔离，也就是想办法把旧的Active NameNode隔离起来，使它不能正常对外提供服务。

在进行fencing的时候，会执行以下的操作：

- 首先尝试调用这个旧的Active NameNode的HAServiceProtocol RPC接口的transitionToStandby方法，看能不能把它转换为Standby状态。
- 如果transitionToStandby方法调用失败，那么就执行Hadoop配置文件之中预定义的隔离措施，Hadoop目前主要提供两种隔离措施，通常会选择sshfence。
  - sshfence：通过SSH登录到目标机器上，执行命令fuser将对应的进程杀死
  - shellfence：执行一个用户自定义的shell脚本来将对应的进程隔离

##### 9. 小文件过多会有什么危害，如何避免？

Hadoop上大量HDFS元数据信息存储在NameNode内存中，因此过多的小文件必定会压垮NameNode的内存。每个元数据对象约占150byte，所以如果有1千万个小文件，每个文件占用一个block，则NameNode大约需要2G空间。如果存储1亿个文件，则NameNode需要20G空间。

显而易见的解决这个问题的方法就是合并小文件，可以选择在客户端上传时执行一定的策略先合并，或者是使用Hadoop的CombineFileInputFormat<K,V>实现小文件的合并。

##### 10. 请说下HDFS的组织架构？

- Client：客户端
  - 切分文件。文件上传HDFS的时候，Client将文件切分成一个一个的Block，然后进行存储
  - 与NameNode交互，获取文件的位置信息
  - 与DataNode交互，读取或者写入数据
  - Client提供一些命令来管理HDFS，比如启动关闭HDFS、访问HDFS目录及内容等想·
- NameNode：名称节点，也称主节点，存储数据的元数据信息，不存储具体的数据
  - 管理HDFS的名称空间
  - 管理数据块（Block）映射信息
  - 配置副本策略
  - 处理客户端读/写请求
- DataNode：数据节点，也称从节点。NameNode下达命令，DataNode执行实际的操作
  - 存储实际的数据块
  - 执行数据块的读/写操作
- Secondary NameNode：并非NameNode的热备。当NameNode挂掉的时候，它并不能马上替换NameNode并提供服务
  - 辅助NameNode，分担其工作量
  - 定期合并FSImage和Edits，并推送给NameNode
  - 在紧急情况下，可辅助恢复NameNode

##### 11. 请说下MR中的Map Task的工作机制？

**简单概述：**

inputFile通过split被切割为多个split文件，通过Record按行读取内容给map（自己写的处理逻辑的方法），数据被map处理完之后交给OutputCollect收集器，对其结果key进行分区（默认使用的hashPartitioner），然后写入buffer，每个map task都有一个内存缓冲区（环形缓冲区），存放着map的输出结果，当缓冲区快满的时候需要将缓冲区的数据以一个临时文件的方式溢写到磁盘，当整个map task结束后再对磁盘中的整个maptask产生的所有临时文件合并，生成最终的正式输出文件，然后等待reduce task的拉取。

**详细步骤：**

- 读取数据组件InputFormat（默认TextInputFormat）会通过getSplits方法对输入目录中的文件进行逻辑切片规划得到block，有多少个block就对应启动多少个MapTask。
- 将输入文件切分为block之后，由RecordReader对象（默认是LineRecordReader）进行读取，以\n作为分隔符，读取一行数据，返回<key,value\>，Key表示每行首字符偏移值，Value表示这一行文本内容。
- 读取block返回<key,value\>，进入用户自己继承的Mapper类中，执行用户重写的map函数，RecordReader读取一行这里调用一次。
- Mapper逻辑结束之后，将Mapper的每条结果通过context.write进行collect数据收集，在collect中，会先对其进行分区处理，默认使用HashPartitioner。
- 接下来，会将数据写入内存，内存中的这边区域叫做环形缓冲区（默认100M），缓冲区的作用是批量收集Mapper结果，减少磁盘IO的影响，我们的Key/Value对以及Partition的结果都会被写入缓冲区，当然，写入之前，Key与Value值都会被序列化成字节数组。
- 当环形缓冲区的数据达到溢写比例（默认0.8），也就是80M时，溢写线程启动，需要对这80MB空间内的Key做排序（Sort）。排序是MapReduce模型默认的行为，这里的排序也是对序列化的字节做的排序。
- 合并溢写文件，每次溢写会在磁盘上生成一个临时文件（写文件判断是否有Combiner），如果Mapper的输出结果真的很大，有多次这样的溢写发生，磁盘上对应的就会有多个临时文件存在，当整个数据处理结束之后开始对磁盘中的临时文件进行Merge合并，因为最终的文件只有一个，写入磁盘，并且为这个文件提供了一个索引文件，以记录每个reduce对应数据的偏移量。

##### 12. 请说下MR中的Reduce Task的工作机制？

**简单概述：**

Reduce大致分为copy、sort、reduce三个阶段，重点在前两个阶段。copy阶段包含一个eventFetcher来获取已完成的map列表，由Fetcher线程去copy数据，在此过程中会启动两个merge线程，分别为inMemoryMerger和onDiskMerger，分别将内存中的数据merge到磁盘和磁盘中的数据进行merge。待数据copy完成之后，copy阶段就完成了，开始进行sort阶段，sort阶段主要是执行finalMerge操作，纯粹的sort阶段，完成之后就是reduce阶段，调用用户定义的reduce函数进行处理。

**详细步骤：**

- Copy阶段：简单地拉取数据。Reduce进程启动一些数据copy线程（Fetcher），通过HTTP方式请求maptask获取属于自己的文件（map task的分区会标识每个map task属于哪个reduce task，默认reduce task的标识从0开始）。
- Merge阶段：这里的merge如map端的merge动作，只是数组中存放的是不同map端copy来的数值。Copy过来的数据会先放入内存缓冲区中，这里的缓冲区大小要比map端的更为灵活。merge有三种形式：内存到内存；内存到磁盘；磁盘到磁盘。默认情况下第一种形式不启动。当内存中的数据量到达一定阈值，就启动内存到磁盘的merge。与map端类似，这也是溢写的过程，这个过程中如果设置了Combiner，也是会启用的，然后在磁盘中生成了众多的溢写文件。第二种merge方式一直在运行，直到没有map端的数据时才结束，然后启动第三种磁盘到磁盘的merge方式生成最终的文件。
- 合并排序：把分散的数据合并成一个大的数据后，还会再对合并后的数据排序。
- 对排序后的键值对调用reduce方法：键相等的键值对调用一次reduce方法，每次调用会产生零个或者多个键值对，最后把这些输出的键值对写入到HDFS文件中。

##### 13. 请说下MR中shuffle阶段？

shuffle阶段分为四个步骤：依次为：分区、排序、规约、分组，其中前三个步骤在map阶段完成，最后一个步骤在reduce阶段完成。shuffle是MapReduce的核心，它分布在MapReduce的map和reduce阶段。一般把从Map产生输出开始到Reduce取得数据作为输入之前的过程称之为shuffle。

- Collect阶段：将MapTask的结果输出到默认大小为100M的环形缓冲区，保存的是key/value，Partition分区信息等。
- Spill阶段：当内存中的数据量达到一定阈值的时候，就会将数据写入本地磁盘，在将数据写入磁盘之前需要对数据进行一次排序操作，如果配置了全局的combiner，还会将有相同分区号和key的数据进行排序。
- Merge阶段：把所有溢出的临时文件进行一次合并操作，以确保一个MapTask最终只产生一个中间数据文件。
- Copy阶段：ReduceTask启动Fetcher线程到已经完成MapTask的节点上复制一份属于自己的数据，这些数据默认会保存在内存的缓冲区汇中，当内存的缓冲区达到一定阈值的时候，就会将数据写到磁盘上。
- Merge阶段：在RecudeTask远程复制数据的同时，会在后台开启两个线程对内存到本地的数据文件进行合并操作。
- Sort阶段：在对数据进行合并的同时，会进行排序操作，由于MapTask阶段已经对数据做了局部的排序，ReduceTask只需要保证Copy的数据的最终整体有序性即可。

> shuffle中的缓冲区大小会影响到MapReduce程序的执行效率，原则上说，缓冲区越大，磁盘IO的次数越少，执行速度就越快。缓冲区的大小可以通过参数调整，参数：`mapreduce.task.io.sort.mb`，默认为100MB。

##### 14. shuffle阶段的数据压缩机制了解吗？

在shuffle阶段，可以看到数据通过大量的拷贝，从map阶段输出的数据，都要通过网络拷贝，发送到reduce阶段，这一过程中，涉及到大量的网络IO，如果数据能够进行压缩，那么数据的发送量就会少的多。

hadoop当中支持的压缩算法：gzip、bzip2、LZO、LZ4、Snappy，这几种压缩算法综合压缩和解压缩的速率，谷歌的Snappy是最优的，一般都选择Snappy压缩。

##### 15. 在写MR时，什么情况下可以使用规约？

规约（combiner）是不能够影响任务的运行结果的，局部汇总，适用于求和类，不适用于求平均值，如果reduce的输入参数类型和输出参数类型是一样的，则规约的类可以使用reduce类，只需要在驱动类中指明规约的类即可。

##### 16. yarn集群的架构和工作原理知道多少？

YARN的基本设计思想是将MapReduce V1中的JobTracker拆分为两个独立的服务：ResourceManager和ApplicationMaster。ResourceManager负责整个系统的资源管理和分配，ApplicationMaster负责单个应用程序的管理。

- ResourceManager：RM是一个全局的资源管理器，负责整个系统的资源管理和分配，它主要由两个部分组成：调度器（Scheduler）和应用程序管理器（Application Manager）。调度器根据容量、队列等限制条件，将系统中的资源分配给正在运行的应用程序，在保证容量、公平性和服务等级的前提下，优化集群资源利用率，让所有的资源都被充分利用。应用程序管理器负责管理整个系统中的所有的应用程序，包括应用程序的提交、与调度器协调资源以启动ApplicationMaster、监控ApplicationMaster运行状态并在失败时重启它。

- ApplicationMaster：用户提交的一个应用程序会对应一个ApplicationMaster，它的主要功能有：

  - 与RM调度器协商以获得资源，资源以Container表示。

  - 将得到的任务进一步分配给内部的任务。

  - 与NN通信以启动/停止任务。

  - 监控所有的内部任务状态，并在任务运行失败的时候重新为任务申请资源以重启任务。

- NodeManager：NodeManager是每个节点上的资源和任务管理器，一方面，它会定期地向RM汇报本节点上的资源使用情况和各个Container的运行状态；另一方面，他接收并处理来自AM的Container的启动和停止请求。

- Container：Container是YARN中的资源抽象，封装了各种资源。一个应用程序会分配一个Container，这个应用程序只能使用这个Container中描述的资源。不同于MapReduce V1中槽位slot的资源封装，Container是一个动态资源的划分单位，更能充分的利用资源。

##### 17. yarn的任务提交流程是怎样的？

当JobClient向YARN提交一个应用程序后，YARN将分两个阶段运行这个应用程序：一是启动ApplicationMaster，第二阶段是由ApplicationMaster创建应用程序，为它申请资源，监控运行直到结束。具体步骤如下：

- 用户向YARN提交一个应用程序，并指定ApplicationMaster程序、启动ApplicationMaster的命令、用户程序。
- RM为这个应用程序分配第一个Container，并与之对应的NM通讯，要求它在这个Container中启动应用程序ApplicationMaster。
- ApplicationMaster向RM注册，然后拆分为内部各个子任务，为各个内部任务申请资源，并监控这些任务的运行，直到结束。
- AM采用轮询的方式向RM申请和领取资源。
- RM为AM分配资源，以Container形式返回。
- AM申请到资源后，便于之对应的NM通讯，要求NM启动任务。
- NodeManager为任务设置好运行环境，将任务启动命令写到一个脚本中，并通过运行这个脚本启动任务。
- 各个任务向AM汇报自己的状态和进度，以便当任务失败时可以重启任务。
- 应用程序完成后，ApplicationMaster向ResourceManager注销并关闭自己。

##### 18. yarn的资源调度三种模型了解吗？

在Yarn中有三种调度器可以选择：FIFO Scheduler，Capacity Scheduler，Fair Scheduler。apache版本的hadoop默认使用的是Capacity Scheduler调度方式。CDH版本的默认使用的是Fair Scheduler调度方式。

**FIFO Scheduler**（先来先服务）：FIFO Scheduler把应用按提交的顺序排成一个队列，这是一个先进先出队列，在进行资源分配的时候，先给队列中最头上的应用分配资源，待最上头的应用需求满足后再给下一个分配，以此类推。FIFO Scheduler是最简单也是最容易理解的调度器，也不需要任何配置，但它并不适用于共享集群。大的应用可能会占用所有集群资源，这就导致其他应用被阻塞，比如有个大的任务在执行，占用了全部的资源，再提交一个小任务，则此小任务会一直被阻塞。

**Capacity Scheduler**（能力调度器）：对于Capacity Scheduler调度器，有一个专门的队列用来运行小任务，但是为小任务专门设置一个队列会预先占用一定的集群资源，这就导致大任务的执行时间会落后于使用FIFO Scheduler调度器的时间。

**Fair Scheduler**（公平调度器）：在Fair Scheduler调度器中，我们不需要预先占用一定的系统资源，Fair调度器会为所有运行的job动态的调整系统资源。比如：当第一个大的job提交时，只有这一个job在运行，此时它获得了所有集群资源，当第二个小任务提交后，Fair调度器会分配一半资源给这个小任务，让两个任务公平的共享集群资源。需要注意的是，在Fair调度器中，从第二个任务提交到获取资源会有一定的延迟，因为它需要等待第一个任务释放占用的Container。小任务执行完成之后也会释放自己占用的资源，大任务又获得了全部的系统资源。最终的效果就是Fair调度器即得到了高的资源利用率又能保证小任务及时完成。

##### 19. Hadoop的优化

###### 1. HDFS小文件的影响

- 影响NameNode的寿命，因为文件元数据存储在NameNode的内存中。
- 影响计算引擎的任务数量，每个小文件都会生成一个Map任务。

###### 2. 数据输入小文件处理

- 合并小文件：对小文件进行归档（Har）、自定义Inputformat将小文件存储成SequenceFile文件。
- 采用ConbinFileInputFormat来作为输入，解决输入端大量小文件场景，对于大量的小文件Job，也可以开启JVM重用。

###### 3. Map阶段

- 增大环形缓冲区大小，由100M扩大到200M。
- 增加环形缓冲区溢写的比例，由80%扩大到90%。

- 减少对溢写文件的merge次数，10个文件，一次20个merge。
- 不影响实际业务的前提下，采用Combiner提前合并，减少I/O。

###### 4. Reduce阶段

- 合理设置Map和Reduce个数：两个都不能太少，也不能太多。太少，会导致Task等待，延长处理事件；太多，会导致Map、Reduce任务间竞争资源，造成处理超时等错误。设置Map、Reduce共存：调整slowstart.completedmaps参数，使Map运行到一定程度后，Reduce也开始运行，减少Reduce的等待时间。
- 规避使用Reduce，因为Reduce在用于连接数据集的时候会产生大量的网络消耗。

- 增加每个Reduce去Map中拿数据的并行数。
- 集群性能可以的前提下，增加Reduce端存储数据内存的大小。

###### 5. IO传输

- 采用数据压缩的方式，减少网络IO的时间。
- 使用SequenceFile二进制文件。

###### 6. 整体

- MapTask默认内存大小为1G，可以增加MapTask内存大小为4
- ReduceTask默认内存大小为1G，可以增加ReduceTask内存大小为4-5g

- 可以增加MapTask的cpu核数，增加ReduceTask的CPU核数
- 增加每个Container的CPU核数和内存大小

- 调整每个Map Task和Reduce Task最大重试次数

##### 20. 如何解决Hadoop的数据倾斜

###### 1. 提前在map进行combine，减少输出的数据量

在Mapper加上combiner相当于提前进行reduce，即把一个Mapper中的相同的key进行了聚合，减少shuffle过程中传输的数据量，以及Reducer端的计算量。

###### 2. 数据倾斜的key，大量分布在不同的mapper

- 局部聚合+全局聚合：第一次在map阶段对那些导致了倾斜的key加上1-n的随机前缀，这样本来相同的key，也会被分到多个Reduce中进行局部聚合，数量就会大大降低。第二次mapreduce，去掉key的随机前缀，进行全局聚合。
- 增加Reducer，提升并行度：JobConf.setNumReduceTasks(int)。

- 实现自定义分区：根据数据分布情况，自定义散列函数，将key均匀分配到不同的Reducer。