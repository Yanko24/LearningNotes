### 3. 垃圾收集器与内存分配策略

#### 3.4 几个概念

##### 3.4.1 吞吐量（Throughput）

处理器用于运行用户代码的时间与处理器总消耗时间的比值，即：
$$
吞吐量 = \frac{运行用户代码时间}{运行用户代码时间 + 运行垃圾收集时间}
$$
如果虚拟机完成某个任务，用户代码加上垃圾收集总共耗费了100分钟，其中垃圾收集花掉1分钟，那么吞吐量就是100%。

##### 3.4.4 并行和并发

在讨论垃圾收集器的上下文语境中，并行和并发可以做如下理解：

###### 3.4.4.1 并行（Parallel）

并行描述的是多条垃圾收集器线程之间的关系，说明同一时间有多条这样的线程在协同工作，通常默认此时用户线程是处于等待状态。

###### 3.4.4.2 并发（Concurrent）

并发描述的是垃圾收集器线程与用户线程之间的关系，说明同一时间垃圾收集器线程与用户线程都在运行。由于用户线程被冻结，所以程序仍然能响应服务请求，但由于垃圾收集器线程占用了一部分系统资源，此时应用程序的处理的吞吐量将受到一定影响。

#### 3.5 经典垃圾收集器

先用一张图，看看这7款经典的垃圾收集器，这7种作用于不同分代的收集器，如果两个收集器之间存在连线，就说明它们可以搭配使用。其中`Serial + CMS`和`ParNew + Serial Old`在JDK9中完全取消了对这些组合的支持。

![](http://yanko.test.upcdn.net/images/7%E7%A7%8D%E7%BB%8F%E5%85%B8%E7%9A%84%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86%E5%99%A8.png)

##### 3.5.1 Serial收集器

Serial收集器是一个单线程收集器，在进行垃圾收集时，必须暂停其他线程的工作，也就是发生“Stop The World”（STW）。在GC期间，应用是不可用的。

![](http://yanko.test.upcdn.net/images/Serial+Serial-Old%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86%E5%99%A8.png)

- 标记-复制算法
- 最基础、最历史悠久的收集器
- 与其他收集器的单线程相比简单而高效，对于内存受限的环境，Serial收集器是额外内存消耗（Memory Footprint）最小的
- 单线程工作的收集器，效率会比极慢，消耗内存小
- 依然是HotSpot虚拟机运行在客户端模式下的默认新生代收集器

##### 3.5.2 ParNew收集器

ParNew收集器实质上是Serial收集器的多线程并行版本，在JDK7之前的系统中首选的新生代收集器，因为它是除了Serial收集器之外，可以与CMS收集器配合工作的收集器。

![](http://yanko.test.upcdn.net/images/ParNew+Serial-Old%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86%E5%99%A8.png)

- 标记-复制算法
- ParNew收集器是激活CMS后（使用`-XX:+UseConcMarkSweepGC`选项）的默认新生代收集器，也可以使用`-XX:+/-UseParNewGC`选项来强制指定或者禁用它
- ParNew收集器默认开启的收集线程数与处理器核心数量相同，可以使用`-XX:ParallelGCThreads`来限制垃圾收集的线程数

##### 3.5.3 Parallel Scavenge收集器

Parallel Scavenge收集器重点关注的目标是达到一个可控制的吞吐量（Throughput），CMS等收集器关注的尽可能是缩短垃圾收集时用户线程的停顿时间，所以Parallel Scavenge收集器也叫做“吞吐量优先收集器”。

![](http://yanko.test.upcdn.net/images/Parallel-Scavenge+Parallel-Old%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86%E5%99%A8.png)

- 标记-复制算法
- 能够并行收集的多线程收集器
- `-XX:MaxGCPauseMillis`控制最大垃圾收集停顿时间，允许的值是一个大于0的毫秒数，收集器尽力保证内存回收花费的时间不超过用户设定值，但注意，并不是越小越好
- `-XX:GCTimeRatio`控制吞吐量的大小，是一个大于0小于100的整数，也就是垃圾收集时间占总时间的比率，相当于吞吐量的倒数。如果此参数为19，则允许最大的垃圾收集时间就占总时间的5%（即1/(1+19)），默认为99
- `-XX:+UseAdptiveSizePolicy`内存自适应开关，开启后不需要人工置顶新生代大小（`-Xmn`）、Eden与Survivor区的比例（`-XX:SurvivorRatio`）以及晋升老年代对象大小（`-XX:PretenureSizeThreshold`）等细节参数，虚拟机会根据当前系统的运行情况收集性能监控信息，动态调整这些参数以提供最合适的停顿时间或者最大的吞吐量

##### 3.5.4 Serial Old收集器

Serial Old是Serial收集器的老年代版本，也是一个单线程收集器。

![](http://yanko.test.upcdn.net/images/Serial+Serial-Old%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86%E5%99%A8.png)

- 标记-整理算法
- 是Serial收集器的老年代版本，也是单线程收集器
- 主要提供客户端模式下的HotSpot虚拟机使用
- 服务端模式下的用途
  - 在JDK5以及之前的版本中与Parallel Scavenge收集器搭配使用
  - 作为CMS收集器发生失败时的后备预案，在并发收集发生“Concurrent Mode Failure“时使用

##### 3.5.5 Parallel Old收集器

Parallel Old是Parallel Scavenge收集器的老年代版本，支持多线程并发收集，在JDK6开始提供。Parallel Old收集器出现后，“吞吐量优先”收集器终于有了比较名副其实的搭配组合，在注重吞吐量或者处理器资源较为稀缺的场合，都可以优先考虑Parallel Scavenge和Parallel Old收集器的组合。需要注意的是Parallel Scavenge收集器架构中本身有PS MarkSweep收集器来进行老年代收集，并非直接调用Serial Old收集器，但是两个实现几乎是一样的。

![](http://yanko.test.upcdn.net/images/Parallel-Scavenge+Parallel-Old%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86%E5%99%A8.png)

- 标记-整理算法
- Parallel Old是Parallel Scavenge收集器的老年代版本
- 支持多线程并发收集

##### 3.5.6 CMS收集器

CMS（Concurrent Mark Sweep）收集器是一种以获取最短回收停顿时间为目标的收集器，如果希望系统停顿时间尽可能短，以给用户带来最好的交互体检，则CMS收集器是非常合适的。CMS在一些公开文档中也被称之为“并发低停顿收集器“（Concurrent Low Pause Collector）。

![](http://yanko.test.upcdn.net/images/Concurrent-Mark-Sweep%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86%E5%99%A8.png)

- 标记-清除算法
- CMS收集器运作过程
  - 初始标记（CMS initial mark）：标记一下GC Roots能直接关联到的对象，速度很快，在垃圾清理线程运行期间需要“Stop The World“
  - 并发标记（CMS concurrent mark）：从GC Roots的直接关联对象开始遍历整个对象图的过程，这个过程耗时较长但是不需要停顿用户线程，可以与垃圾收集线程一起并发运行
  - 重新标记（CMS remark）：为了修正并发标记期间因用户程序继续运行而导致标记产生变动的那一部分标记记录，这个阶段也需要“Stop The World“，比初始标记阶段时间稍长，但远比并发标记阶段的时间短
  - 并发清楚（CMS concurrent sweep）：清理删除掉标记阶段判断的已经死亡的对象，可以和用户线程同时并发

- 以获取最短回收停顿时间为目标的收集器
- 缺点
  - 对处理器资源非常敏感，CMS收集器在并发阶段虽然不会导致用户线程停顿，但因为占用了一部分线程而导致应用程序变慢，降低吞吐量。CMS默认启动的回收线程数是`(处理器核心数量 + 3)/4`
  - 无法处理“浮动垃圾”（Floating Garbage），有可能出现“Concurrent Mode Failure“失败进而导致另一次完全“Stop The World“的Full GC的产生。因为垃圾收集阶段用户线程还在持续运行，所以必须要预留足够内存空间提供给用户线程使用，在JDK5的设置下，CMS收集器当老年代使用了68%的空间后就会被激活，在JDK6中，这个阈值已经提升到了92%，可以通过参数`-XX:CMSInitiatingOccupancyFraction`来调整该阈值。如果阈值设置太高，当CMS收集器运行期间，预留的内存不足以满足程序分配新对象的需要，就会出现一次“并发失败”（Concurrent Mode Failure），这时虚拟机启动备用预案，冻结用户线程的执行，临时启用Serial Old收集器来进行老年代的垃圾收集
  - 在收集结束时会产生大量的空间碎片，当无法找到连续的空间分配给当前大对象使用时，不得不触发Full GC。为了解决此问题，引入了两个参数
    - `-XX:+UseCMSCompactAtFullCollection`：默认开启，从JDK9开始废弃。用于在CMS收集器不得不进行Full GC时开启内存碎片的合并整理过程，由于内存整理必须移动活对象，是无法并发的，会导致停顿时间变长
    - `-XX:CMSFullGCsBeforeCompaction`：默认为0，这个参数的作用要求CMS收集器在执行若干次（数量由参数值决定）不整理空间的Full GC之后，下一次进入Full GC之前会先进行碎片整理

##### 3.5.7 Garbage First收集器

Garbage First（简称G1）开创了收集器面向局部收集的设计思路和基于Region的内存布局形式。G1面向堆内存任何部分来组成回收集（Collection Set，一般简称CSet）进行回收，衡量标准是那块内存存放的垃圾数量最多，回收收益最大，这也是G1独有的Mixed GC模式。

G1基于Region的内存布局是实现Mixed GC的关键，G1不再坚持固定大小以及固定数量的分代区域划分，而是把连续的Java堆划分成多大大小相等的独立区域（Region），每一个Region都可以根据需要扮演新生代的Eden空间、Survivor空间或者老年代空间，收集器能够对扮演不同角色的Region采用不同的策略去处理。Region有一部分特殊的Humongous区域，专门用来存储大对象。G1认为只要超过了一个Region容量一半的对象即为大对象，可以通过`-XX:G1HeapRegionSize`设置Region大小，范围是1MB~32MB，且应为2的N次幂。对于那些超过了整个Region容量的超级大对象，将会被存放在N个连续的Humongous Region之中，G1的大多数行为都把Humongous Region作为老年代的一部分来看待。G1收集器的Region分区如下所示：

![](http://yanko.test.upcdn.net/images/G1%E6%94%B6%E9%9B%86%E5%99%A8Region%E5%88%86%E5%8C%BA.png)

G1收集器把堆划分为大小相同的Region，每个Region都会扮演一个角色，分别是E、S、H、O。

- E代表Eden空间
- S代表Survivor空间
- H代表Humongous空间
- O代表Old空间

G1保留了新生代和老年代的概念，但是新生代和老年代不再是固定的，它们是一系列区域（不需要连续）的动态集合。G1收集器之所以能够建立可预测的停顿时间模型，是因为它将Region作为单次回收最小的单元，即每次收集到的空间都是Region大小的整数倍，这样可以避免在整个Java堆中进行全区域的垃圾收集。实现思路是让G1收集器去跟踪各个Region里面的垃圾堆积的“价值”大小，价值即回收所获得的空间大小以及回收所需时间的经验值，然后在后台维护一个优先级列表，每次根据用户设定允许的停顿收集时间（参数`-XX:MaxGCPauseMillis`指定，默认为200毫秒），优先处理回收那些价值收益最大的Region列表。

![](http://yanko.test.upcdn.net/images/G1%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86%E5%99%A8.png)

- 整理基于“标记-整理”算法，局部（两个Region之间）基于“标记-复制”算法
- G1收集器运作过程，除了并发标记外，其他阶段也要完全暂停用户线程
  - 初始标记（Initial Marking）：仅仅只是标记一下GC Roots能直接关联到的对象，并且修改TAMS指针的值，让下一阶段用户线程并发运行时，能正确的在可用的Region中分配对象。这个阶段需要停顿线程，但耗时很短，而且是借用Minor GC的时候同步完成的，所以G1收集器在这个阶段实际并没有额外的停顿
  - 并发标记（Concurrent Marking）：从GC Roots开始对堆中对象进行可达性分析，递归扫描整个堆里的对象图，找出要回收的对象，这阶段耗时很长，但可与用户线程并发执行。当对象图扫描完成后，还要重新处理SATB记录下的在并发时有引用变动的对象
  - 最终标记（Final Marking）：对用户线程做一个短暂的暂停，用于处理并发阶段后仍遗留下来的最后那少量的SATB记录
  - 筛选回收（Live Data Counting and Evacuation）：负责更新Region的统计数据，对各个Region的回收价值和成本进行排序，根据用户所期望的停顿时间来制定回收计划，可以自由选择任意多个Region构成回收集，然后把决定回收的那一部分的Region中的存活对象复制到空的Region中，再清理掉整个旧Region的全部空间。这里的操作涉及活对象的移动，必须暂停用户线程，由多条收集器线程并行完成
- 特点：
  - 并行与并发：G1能充分利用多CPU、多核环境下的硬件优势，可以通过并发的方式让用户程序继续运行，用并行的形式提高收集所需的时间，进一步缩短STW的时间
  - 分代收集：分代概念在G1中仍有保留，它能够采用不同的策略去处理扮演不同角色的Region，无论是新创建的对象还是已经存活一段时间、熬过多次收集的对象都能获取很好的收集效果
  - 空间整合：G1从整理上看基于“标记-整理”算法，从局部看基于“标记-复制”算法，G1运行期间不会产生内存空间碎片
  - 可预测停顿：G1比CMS厉害在能建立可预测的停顿时间模型，能让使用者明确在一个长度为M毫秒的时间片段内，消耗在垃圾收集上的时间大概率不会超过N毫秒
  - 目前小内存应用CMS的表现大概率仍然要优于G1，这个优劣势的Java堆容量平衡点通常在6GB到8GB之间