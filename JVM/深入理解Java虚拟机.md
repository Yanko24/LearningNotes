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

##### 3.5.7 Garbage First收集器