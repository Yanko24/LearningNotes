#### Zookeeper常见的面试题

##### 1. 什么是Zookeeper？谈谈你对Zookeeper的认识？

Zookeeper是一个分布式的，开放源代码的分布式应用程序协调服务。它是一个为分布式应用提供一致性服务的软件，提供的功能包括：配置维护、域名服务、分布式同步、组服务等。

##### 2. Zookeeper的核心功能？

Zookeeper提供了三个核心功能：文件系统、通知机制和集群管理机制。

###### 1. 文件系统

Zookeeper存储数据的结构，类似于一个文件系统。每个节点称之为znode，买个znode都是类似于K-V的结构，每个节点的名字相当于key，每个节点中都保存了对应的数据，类似于key-value中的value。

###### 2. 通知机制

当某个client监听某个节点时，当该节点发生变化时，zookeeper就会通知监听该节点的客户端，后续根据客户端的处理逻辑进行处理。

###### 3. 集群管理机制

zookeeper本身是一个集群结构，有一个leader节点，负责写请求，多个follower节点负责相应读请求。并且在leader节点故障的时候，会根据选举机制从剩下的follower中选举出新的leader。

##### 3. Zookeeper中的角色都有哪些？

###### 1. leader

处理所有的事务请求（写请求），可以处理读请求，集群中只能有一个leader。

###### 2. follower

只能处理读请求，同时作为leader的候选节点，即如果leader宕机，follower节点要参与到新的leader选举中，有可能成为新的leader节点。

###### 3. observer

只能处理读请求，不能参与选举。

##### 4. Zookeeper中的节点有哪几种，分别有什么不同？

Zookeeper中的节点一共分为7中，分别是持久节点（PERSISTENT）、持久顺序节点（PERSISTENT_SEQUENTIAL）、临时节点（EPHEMERAL）、临时顺序节点（EPHEMERAL_SEQUENTIAL）、容器节点（CONTAINER）、带过期时间的持久节点（PERSISTENT_WITH_TTL）、带过期时间的持久顺序节点（PERSISTENT_SEQUENTIAL_WITH_TTL）。

###### 1. 持久节点（PERSISTENT）

持久节点，一旦创建成功不会被删除，除非客户端主动发起删除请求。

###### 2. 持久顺序节点（PERSISTENT_SEQUENTIAL）

持久顺序节点，会在用户路径后面拼接一个不会重复的自增数字后缀，一旦创建成功不会被删除，除非客户端主动发起请求。

###### 3. 临时节点（EPHEMERAL）

临时节点，当创建该节点的客户端断开连接后就会被自动删除。

###### 4. 临时顺序节点（EPHEMERAL_SEQUENTIAL）

临时顺序节点，创建时会在用户路径后面拼接一个不会重复的自增数字后缀，当创建该节点的客户端断开连接后就会被自动删除。

###### 5. 容器节点（CONTAINER）

容器节点，一旦子节点被删除完，该节点就会被服务端自动删除。

###### 6. 带过期时间的持久节点（PERSISTENT_WITH_TTL）

带过期时间的持久节点，带有超时时间的节点，如果超时时间内没有子节点被创建，就会被删除。需要开启服务端配置`extendedTypesEnabled=true`。

###### 7. 带过期时间的持久顺序节点（PERSISTENT_SEQUENTIAL_WITH_TTL）

带过期时间的持久顺序节点，创建节点时会在用户路径后面拼接一个不会重复的自增数字后缀，带有超时时间，如果超时时间内没有子节点被创建，就会被删除。需要开启服务端配置`extendedTypesEnabled=true`。

##### 5. Zookeeper的工作原理

Zookeeper的核心是原子广播，这个机制保证了各个Server之前的同步。实现这个机制的协议叫做Zab协议。Zab协议有两种模式，分别是恢复模式（选主）和广播模式（同步）。当服务启动或者在leader崩溃后，Zab就进入了恢复模式，当leader被选举出来，且大多数Server完成了和leader的状态同步后，恢复模式就结束了。状态同步保证了leader和server具有相同的系统状态。之后进入广播模式，如果这个时候当一个server加入到Zookeeper服务中，它会在恢复模式下启动，发现leader，并和leader进行状态同步。等同步结束，它也参与消息广播。Zookeeper服务一直维持在Broadcast状态，直到leader崩溃或者leader失去了大部分followers的支持。

##### 6. Zookeeper是如何保证事务的顺序一致性的？

Zookeeper采用了递增的事务Id来标识，所有的proposal（提议）都在被提出时加上了zxid，zxid实际上是一个64位的数字，高32位是epoch（纪元）用来标识leader是否发生改变，如果有新的leader产生出来，epoch会自增，低32位用来递增计数。当新产生proposal时，会依赖数据库的两阶段过程，首先会向其他的server发出事务执行请求，如果超过半数的机器都能执行并且能够成功，那么久开始执行。

##### 7. Zookeeper下的server的工作状态？

每个Server在工作过程中有三种状态：

- LOKKING：竞选状态，当前Server不知道leader是谁，正在搜寻
- LEADER：领导者状态，当前Server即为选举出来的leader
- FOLLOWING：随从状态，leader已经选举出来，当前Server与之同步
- OBSERVING：观察状态，同步leader状态，不参与投票

##### 8. 集群中为什么需要leader？

在分布式环境中，有些业务逻辑只需要集群中的某一台机器执行，其他的机器可以共享这个结果，这样可以大大减少重复计算，提高性能，于是就需要进行leader选举。

##### 9. Zookeeper节点宕机怎么处理？

Zookeeper本身也是集群，推荐配置不少于3个服务。Zookeeper自身也要保证当一个节点宕机时，其他节点会继续提供服务。

如果是一个Follower宕机，还有2台服务器提供访问，因为Zookeeper上的数据是有多个副本的，数据并不会丢失；如果是一个Leader宕机，Zookeeper会选举出新的Leader。

Zookeeper集群的机制是只要超过半数的节点正常，集群就能正常提供服务。只有在Zookeeper节点挂的太多，只剩一半或者一半不到的节点能工作时，集群才会失效。

##### 10. Zookeeper是如何选择leader节点的？

当leader崩溃或者leader失去大多数的follower，这时Zookeeper集群进入恢复模式，恢复模式需要重新选举一个leader，让所有的Server都恢复到一个正确的状态。Zookeeper的选举算法有两种：一种是基于`basic paxos`实现的，另一种是基于`fast paxos`算法实现的。系统默认的选举算法为`fast paxos`。Zookeeper集群选举leader节点，有两种情况：①集群刚刚启动时；②当原有的leader节点崩溃时。

部分名词解释：

```
服务器id：myid（或sid，集群模式下必有该配置项）
事务id：服务器中存放的最大zxid
逻辑时钟：发起的投票轮数计数，用来怕判断多个投票是否在同一轮选举周期中，该值在服务端是一个自增序列。
```

无论对于那种情况选举leader，只要逻辑时钟不是最新的，这个Server的投票会被清除。

###### 1. 服务器启动时期的Leader选举

- 每个Server发出一个投票。由于是初始情况，Server1和Server2都会将自己作为leader服务器来进行投票，每次投票会包含锁推举的服务器的myid和zxid，使用（myid, zxid）表示，此时Server1的投票为（1, 0），Server2的投票为（2, 0），然后各自将这个投票发给集群中其他机器。

- 接受来自各个服务器的投票。集群的每个服务器收到投票后，首先判断该投票的有效性，如检查逻辑时钟、是否来自于LOOKING状态的服务器。

- 处理投票。针对每一个投票，服务器都需要将别人的投票和自己的投票进行比较，比较规则如下：

  ```
  优先判断zxid。zxid（事务ID）比较大的服务器优先作为Leader。
  如果zxid相同，那么久比较myid。myid（服务器ID）比较大的服务器作为Leader服务器。
  ```

对于Server1而言，它的投票是（1, 0），接收Server2的投票为（2, 0），首先会比较两者的zxid，均为0，在比较myid，此时Server2的myid最大，于是更新自己的投票为（2, 0），然后重新投票，对于Server2而言，不需要更新自己的投票，只是再次向集群中所有机器发出上一次投票信息即可。

- 统计投票。每次投票后，服务器都会统计投票信息，判断是否已经有过半机器接收到相同的投票信息，对于Server1、Server2恶言，都统计出集群中已经有两台机器接受了（2, 0）的投票信息，此时便认为已经选出了Leader。
- 改变服务器状态。一旦确定了Leader，每个服务器就会更新自己的状态，如果是Follower，那么就变更为FOLLOWING，如果是Leader，那就变更为LEADING。

###### 2. 服务器运行时期Leader选举

在Zookeeper运行期间，Leader与非Leader服务器各司其职，即便当有非Leader服务器宕机或者新加入，此时也不影响Leader，但是一旦Leader服务器挂了，那么整个集群将暂停对外服务，进入新一轮Leader选举，其过程和启动时期的Leader选举过程基本一致。假设正在运行的有Server1、Server2、Server3三台服务器，当前Leader是Server2，若某一时刻Leader挂了，此时便开始Leader选举。

- 变更状态。Leader宕机后，剩下的非Observer服务器都会将自己的服务器状态变更为LOOKING，然后开始进入Leader选举过程。
- 每个Server发出一个投票。在运行期间，每个服务器上的zxid可能不同，此时假定Server1的zxid为111，Server3的zxid为122；在第一轮投票中，Server1和Server2都会投子，产生投票（1, 111)，（3, 122），然后将各自投票发送给集群中所有机器。
- 接收来自各个服务器的投票。与启动时过程相同。
- 处理投票。与启动时过程相同，此时，Server3会成为Leader。
- 统计投票。与启动过程相同。
- 改变服务器状态。与启动过程相同。