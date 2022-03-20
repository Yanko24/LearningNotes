#### Kafka常见的面试题

##### 1. 为什么要使用 kafka？

##### 2. Kafka的组件与架构？

- Producer：消息生产者，就是向Kafka Broker发消息的客户端（Push）
- Consumer：消息消费者，向Kafka Broker取消息的客户端（Pull）
- Consumer Group（CG）：消费者组，由多个consumer组成。消费者组内每个消费者负责消费不同分区的数据，一个分区只能由一个组内消费者消费；消费者组之间互不影响。所有的消费者都属于某个消费者组，即消费者组是逻辑上的一个订阅者
- Broker：一台Kafka服务器就是一个Broker。一个集群由多个Broker组成。一个Broker可以容纳多个Topic
- Topic：可以理解为一个队列，生产者和消费者面向的都是一个Topic
- Partition：为了实现扩展性，一个非常大的topic可以分布到多个broker（即服务器）上，一个topic可以分为多个partition，每个partition是一个有序的队列
- Replica：副本，为保证集群中的某个几点发生故障时，该节点上的partition数据不丢失，且Kakfa仍然能继续工作，kafka提供了副本机制，一个topic的每个分区都有若干个副本，一个leader和若干个follower
- Leader：每个分区多个副本的“主”，生产者发送数据的对象，以及消费者消费数据的对象都是leader
- Follower：每个分区多个副本中的“从”，实时从leader中同步数据，保持和leader数据的同步。leader发生故障时，某个follower会成为新的leader

##### 3. Kafka为什么要分区？

- 方便在集群中扩展，每个Partition可以通过调整以适应它所在的机器，而一个topic又可以有多个Partition组成，因此整个集群就可以适应任意大小的数据
- 可以提高并发，因为可以以Parition为单位读写

##### 4. Kafka生产者分区策略？

- 指明partition的情况下，直接将指明的值直接作为partition值
- 没有指明partition值但是是key的情况下，将key的hash值与topic的partition数进行取余得到partition值
- 既没有partition值又没有key值的情况下，第一次调用时随机生成一个整数（后面每次调用在这个整数上自增），将这个值与topic可用的partition总数取余得到partition的值，也就是常说的round-robin算法

##### 2. Kafka消费过的消息如何再消费？

Kafka消费消息的offset是定义在zookeeper中的，如果想重复消费Kafka的消息，可以在redis中自己记录offset的checkpoint点（n个），当想重复消费消息时，通过读取redis中的checkpoint点进行zookeeper的offset重设，这样就可以达到重复消费消息的目的了。

##### 3. kafka的数据是放在磁盘上还是内存上，为什么速度会快？

- 顺序写入：因为硬盘是机械结构，每次读写都会寻址→写入，其中寻址是一个“机械动作”，它是非常耗时的。所以硬盘讨厌随机I/O，喜欢顺序I/O。为了提高读写硬盘的速度，Kafka就是使用的顺序I/O。
- 内存映射文件：64位操作系统中一般可以表示20G的数据文件，它的工作原理是直接利用操作系统的Page来实现文件到物理内存的直接映射。完成映射之后对物理内存的操作会被同步到硬盘上。
- Kafka高效文件存储设计：Kafka把一个Topic中的一个Partition大文件分成多个小文件段，通过多个小文件段，就容易定期清除或删除已经消费完的文件，减少磁盘占用；通过索引信息可以快速定位message和response的最大大小；通过index元数据全部映射到memory（内存映射文件），可以避免segment file的IO磁盘操作；通过索引文件稀疏存储，可以大幅度降低index文件元数据占用的空间大小。

##### 4. Kafka的topic,broker,partition之间的关系？

一个topic对应多个partition，partition分布在多broker上，多broker一起提供kafka服务。Kafka中，Topic是一个存储消息的逻辑概念，可认为是一个消息的集合。物理上，不同Topic的消息分开存储，每个Topic可划分多个partition，同一个Topic下的不同的partition包含不同消息。每个消息被添加至分区时，分配唯一offset，以此保证partition内消息的顺序性。

##### 5. 如何保证Kafka中数据的有序性？

- 全局有序：1个Topic对应1个Partition，使用1个Consumer消费
- 局部有序：在发送消息的时候指定Partition Key，这样Partition Key相同的就会在1个分区，使用1个Consumer消费1个Partition。

##### 6. Kafka中的Leader是指什么？

Kafka集群的Leader有两个含义：一个是Kafka集群节点Broker的Leader节点；一个是Topic分区的Leader。

###### 1. Broker的Leader

Kafka集群的Broker的Leader的选举算法没有Zookeeper那么复杂，Kafka的Leader选举是通过在zookeeper上创建`/controller`临时节点来实现leader选举，并在该节点中写入当前broker的信息`{"version":1,"brokerid":3,"timestamp":"1645927025307"}`，利用Zookeeper的强一致性特性，一个节点只能被一个客户端创建成功，创建成功的broker即为leader，即先到先得原则，leader也就是集群中的controller，负责集群中的大小事务。

- 创建、删除主题，增加分区并分配leader分区
- 集群Broker管理（新增Broker、Broker主动关闭、Broker故障）
- preferred leader的选举（集群Topic的所有副本的第一个副本叫做preferred replica，刚刚创建的topic一般preferred replica就是preferred leader）
- 分区重分配

###### 2. Partition的Leader

首先Kafka会将接受到的消息分区（Partition），每个主题（Topic）的消息有不同的分区。这样一方面消息的存储就不会受到单一服务器存储空间大小的限制，另一方面消息的处理也可以在多个服务器上并行。其次是为了保证高可用，每个分区都会有一定量的副本（Replica）。这样如果有部分服务器不可用，副本所在的服务器就会接替上来，保证应用的持续性。但是，为了保证较高的处理效率，消息的读写都是在固定的一个副本上完成。这个副本就是所谓的Leader，而其他副本则是Follower，而Follower则会定期地到Leader上同步数据。

分区Leader的选举：如果某个分区所在的服务器出了问题，不可用，Kafka会从该分区的其他的副本中选择一个作为新的Leader。之后所有的读写就会转移到这个新的Leader上。Kafka会在Zookeeper上针对每个Topic维护一个称为ISR（in-sync replica，已同步的副本）的集合，该集合中是一些分区的副本。只有当这些副本都跟Leader中的副本同步了之后，Kafka才会认为消息已经提交，并反馈给消息的生产者。如果这个集合有增减，Kafka会更新Zookeeper上的记录。如果某个分区的Leader不可用，Kafka就会从ISR集合中选择一个副本作为新的Leader。

##### 7. Kafka数据怎么保障不丢失？

Kafka数据不丢失，一共分三个方面，生产者端、消费者端和Broker端。

###### 1. 生产者数据的不丢失

Kafka的ACK机制：在Kafka发送数据的时候，每次发送消息都会有一个确认反馈机制，确保消息正常的能够被收到，其状态有-1，0，1。

- 同步模式（sync），参数`producer.type`

  ack设置为0，风险很大，一般不建议设置为0。即使设置为1，也会苏浙leader宕机丢失数据。所以如果要严格保证生产者端数据不丢失，可设置为-1。

- 异步模式（async），参数`producer.type`

  也会考虑ack的状态，除此之外，异步模式下还有一个buffer，通过buffer来进行控制数据的发送，有两个值来进行控制，时间阈值与消息的数量阈值，如果buffer满了数据还没有发送出去，有个选线是配置是否立即清空buffer。可以设置为-1，永久阻塞，也就是数据不再生产。异步模式下，即使设置为-1，也可以能因为程序猿的操作问题导致数据丢失，比如kill -9。

```
ack=0: producer不等待broker同步完成的确认，继续发送下一条（批）消息
ack=1: 默认的ack状态，producer要等待leader成功收到数据并得到确认，才发送下一条消息
ack=-1: producer得到follower确认，才发送下一条消息。当min.insync.replicas（配置的备份个数）为1是，和ack=1没有区别
```

除了ACK机制外，生产者还可以配置消息重试机制来保证Kafka数据的不丢失。

```shell
# retries的参数意义：当消息发送失败时，客户端会进行重试，retries表示重试的次数
retries参数的默认值为Interger.MAX_VALUE
# 如果max.in.flight.requests.per.connection参数不为1，则可能因为重试导致消息的顺序发生改变
max.in.flight.requests.per.connection的默认值是5
```

根据以上信息，为了保证生产者数据不丢失，可以有以下几种配置：

```shell
# 高可用
ack=all, retries > 0(默认Integer.MAX_VALUE), retry.backoff.ms = 100ms(默认)
优点：这样可以保证producer发送的每一条数据都要成功，如果不成功会将消息缓存起来，等恢复后再进行发送。
缺点：保证了高可用，但是集群的吞吐量不是很高，数据发送到broker之后，leader要将数据同步到follower上，受其他的影响，ack响应时间可能会更长

# 折中
ack=1, retries > 0(默认Integer.MAX_VALUE), retries时间间隔设置
优点：保证了消息的可靠性和吞吐量
缺点：性能处于两者中间

# 高吞吐
ack=0
优点：可以容忍数据的丢失，吞吐量大，可以接受大量请求
缺点：不知道发送的消息是否成功
```

###### 2. 消费者数据的不丢失

通过Offset commit来保证数据的不丢失，Kafka自己记录了每次消费的offset数值，下次继续消费的售后，会接着上次的offset进行消费。而offset信息在kafka0.9版本之前保存在Zookeeper中，在0.9之后保存在topic中（__consumer_offsets），即使消费者在运行过程中挂掉了，再次启动的时候也会找到offset的值，找到之前消费消息的位置，接着消费，由于offset的信息写入的时候并不是每条消息消费完成之后都写入的。所以这种情况有可能会造成重复消费，但是不会丢失数据。

正常情况下消费者是自动提交offset的，可以考虑修改参数`enable.auto.commit`为false，手动提交offset，当确认客户端对这个消息处理完成后，自己提交offset。

###### 3. Kafka集群中的Broker的数据不丢失

每个Broker中的partition一般都会设置有replication（副本）的个数，生产者写入的时候首先根据分发策略（有partition按partition，有key按key，都没有轮询）写入到leader中，follower（副本）再跟leader同步数据，这样有了备份，也可以保证数据的不丢失。

##### 8. 采集数据为什么选择kafka？



##### 9. kafka 重启是否会导致数据丢失？

Kafka是将数据写到磁盘的，一般数据不会丢失。但是在重启Kafka过程中，如果有消费者消费消息，那么Kafka如果来不及提交offset，可能会造成数据的不准确（丢失或者重复消费）。

##### 10. kafka 宕机了如何解决？

- 先考虑业务是否受到影响

  Kafka宕机了，首先考虑的问题应该是所提供的服务是否因为宕机的机器而受到影响，如果服务提供没问题，如果事先做好了集群的容灾机制，就不需要担心了。

- 节点排错与恢复

  想要恢复集群的节点，主要的步骤就是通过日志分析来查看节点宕机的原因，从而解决，重新恢复节点。

##### 11. 为什么Kafka不支持读写分离？

在Kafka中，生产者写入消息、消费者读取消息的操作都是与Leader副本进行交互的，从而实现的是一种主写主读的生产消费模型。Kafka不支持主写从读，因为主要主写从读有2个缺点：

- 数据一致性问题：数据从主节点转到从节点必然会有一个延时的时间窗口，这个时间窗口会导致主从节点之间的数据不一致。
- 延时问题：类似Redis这种组件，数据从写入主节点到同步到从节点中的过程需要经历`网络→主节点内存→网络→从节点内存`这几个阶段，整个过程会耗费一定的时间。而在Kafka中，主从同步会比Redis更加耗时，它需要经历`网络→主节点内存→主节点磁盘→网络→从节点内存→从节点磁盘`这几个阶段。对延时敏感的应用而言，主写从读的功能并不太适用。

相比来看，Kafka主写主读的优点：

- 可以简化代码的实现逻辑，减少出错的可能
- 将负载粒度细化均摊，与主写从读相比，不仅负载效能更好，而且对用户可控（负载粒度是partition leader）
- 没有延时的影响
- 在副本稳定的情况下，不会出现数据不一致的情况

##### 12. kafka数据分区和消费者的关系？

每个分区只能由同一个消费者组内的一个消费者（consumer）来消费，可以由不同的消费者组的消费者来消费，同组的消费者则起到并发的效果。

##### 13. kafka的数据offset读取流程？

- 连接Zookeeper集群，从Zookeeper中拿到对应topic的partition信息和partition的leader的相关信息
- 连接到对应leader对应的broker
- consumer将自己已经保存的offset发送给leader
- leader根据offset等信息定位到segment（索引文件和日志文件）
- 根据索引文件中的内容，定位到日志文件中该偏移量对应的开始位置读取相应长度的数据并返回给consumer

##### 14. kafka内部如何保证顺序，结合外部组件如何保证消费者的顺序？

Kafka只能保证partition内是有序的，但是partition间的有序是没办法保证的。

##### 15. Kafka消息数据积压，Kafka消费能力不足怎么处理？

- 如果是Kafka消费能力不足，则可以考虑增加Topic的分区数，并且同时提升消费者组的消费者数量，消费者数=分区数
- 如果是下游的数据处理不及时，提高每批次拉取的数量。批次拉取数据过少（拉取数据/处理时间<生产速度），使处理的数据小于生产的数据，也会造成数据积压

##### 16. Kafka单条日志传输大小？

Kafka对于消息体的消息默认认为单条最大值是1M，但是在我们应用场景中，常常会出现一条消息大于1M，如果不对Kafka进行配置。则会出现生产者无法将消息推送到Kafka或者消费者无法去消费Kafka里面的数据，这时我们就要对kafka的server文件（`server.properties`）进行以下配置：

```shell
# broker可复制的消息的最大字节数，默认为1M
replica.fetch.max.bytes: 1048567

# kafka会接收单个消息size的最大限制，默认为1M左右
message.max.bytes: 1000012
```

注意：`message.max.bytes`必须小于等于`replica.fetch.max.bytes`，否则就会导致replica之间数据同步失败。