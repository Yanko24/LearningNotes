### Spark的运行环境

#### 1. Spark的运行环境_Local

Spark作为一个数据处理框架和计算引擎，被设计在所有常见的集群环境中运行，在国内工作中主流的环境为Yarn。

![](http://typora-image.test.upcdn.net/images/Spark运行环境.jpg)

所谓的Local模式，就是不需要其他任何节点资源就可以在本地执行Spark代码的环境，一般用于教学、调试、演示等。

##### 1. 启动Local环境

- 进入解压缩后的路径，执行如下命令

  ```
  bin/spark-shell
  ```

  ![](http://typora-image.test.upcdn.net/images/local.jpg)

- 启动成功后，可以输入网址进行Web UI监控页面访问

  ```
  http://虚拟机地址:4040
  ```

  ![](http://typora-image.test.upcdn.net/images/Spark-Web-UI.jpg)

##### 2. 命令行工具

- 在解压缩的目录下的data目录下，添加words.txt文件。在命令行工具中执行如下代码

  ```
  sc.textFile("data/words.txt").flatMap(_.split(" ")).map((_,1)).reduceByKey(_+_).collect
  ```

  ![](http://typora-image.test.upcdn.net/images/wordcount.jpg)

##### 3. 退出本地模式

```
:quit
```

##### 4. 提交应用

```
bin/spark-submit \
--class org.apache.spark.examples.SparkPi \
--master local[2] \ 
./examples/jars/spark-examples_2.11-2.4.6.jar \
10
```

- --class 表示要执行程序的主类
- --master local[2] 部署模式，默认为本地模式，数字表示分配的虚拟CPU核数量
- spark-examples_2.11-2.4.6.jar 运行的应用类所在的jar包
- 数字10表示程序的入口参数，用于设定当前应用的任务数量

![](http://typora-image.test.upcdn.net/images/spark-shell-pi.jpg)

#### 2. Spark的运行环境_Standalone

只使用Spark自身节点运行的集群模式，也就是我们所谓的独立部署（Standalone）模式。Spark的Standalone模式体现了经典的master-slave模式。

##### 1. 解压缩文件

将spark-2.4.6文件上传到CetnOS并解压在指定位置

##### 2. 修改配置文件

- 进入解压缩后的路径的conf目录，修改slaves.template文件名为slaves

  ```
  mv slaves.template slaves
  ```

- 修改salves文件，添加work节点

  ```
  master
  slave1
  slave2
  ```

- 修改spark-env.sh.template文件名为spark-env.sh

  ```
  mv spark-env.sh.template spark-env.sh
  ```

- 修改spark-env.sh文件，添加以下内容

  ```shell
  export JAVA_HOME=/opt/apps/jdk1.8.0_162
  export SCALA_HOME=/opt/apps/scala-2.11.12
  SPARK_MASTER_HOST=master
  SPARK_MASTER_PORT=7077
  SPARK_WORKER_MEMORY=1024MB
  HADOOP_CONF_DIR=/opt/apps/hadoop-2.7.7/etc/hadoop
  
  SPARK_MASTER_WEBUI_PORT=8888
  ```

- 将spark分发到集群的每台机器中

  ```
  scp spark-2.4.6 slave1:/opt/apps
  scp spark-2.4.6 slave2:/opt/apps
  ```

##### 3. 启动集群

- 指定脚本命令

  ```
  sbin/start-all.sh
  ```

  ![](http://typora-image.test.upcdn.net/images/start-all.jpg)

- 查看Master资源监控Web UI界面

  ![](http://typora-image.test.upcdn.net/images/master.jpg)

##### 4. 提交应用

```
bin/spark-submit \
--class org.apache.spark.examples.SparkPi \
--master spark://master:7077 \
./examples/jars/spark-examples_2.11-2.4.6.jar \
10
```

- --class 表示要执行程序的主类
- --master spark://master:7077 独立部署模式，连接到Spark集群
- spark-examples_2.11-2.4.6.jar 运行类所在的jar包
- 数字10表示程序的入口参数，用于设定当前应用的任务数量

![](http://typora-image.test.upcdn.net/images/standalone提交作业.jpg)

执行任务时，会产生多个Java进程

![](http://typora-image.test.upcdn.net/images/spark提交任务.jpg)

执行任务时，默认采用服务器集群节点的总核数，每个节点内存1024MB

![spark提交作业](http://typora-image.test.upcdn.net/images/spark提交作业.jpg)

##### 5. 提交参数说明

在提交应用中，一般会同时提交一些参数：

| 参数                     | 解释                                                         | 可选值举例                                |
| :----------------------- | ------------------------------------------------------------ | ----------------------------------------- |
| --class                  | Spark程序中包含主函数的类                                    |                                           |
| --master                 | Spark程序运行的模式（环境）                                  | 模式：local[*]、spark://master:7077、Yarn |
| --executor-memory 1G     | 指定每个executor可用内存为1G                                 | 符合集群内存配置即可，具体情况具体分析    |
| --total-executor-cores 2 | 指定所有executor使用的cpu核数为2个                           | 符合集群内存配置即可，具体情况具体分析    |
| --executor-cores         | 指定每个executor使用的cpu核数                                | 符合集群内存配置即可，具体情况具体分析    |
| application-jar          | 打包好的应用jar，包含依赖。这个URL在集群中全局可见。比如hdfs://共享存储系统，如果是file://path，南所有的节点path都要包含同样的jar | 符合集群内存配置即可，具体情况具体分析    |
| application-arguments    | 传给main()方法的参数                                         | 符合集群内存配置即可，具体情况具体分析    |

##### 6. 配置历史服务

由于spark-shell停止掉后，集群监控`master:4040`页面就看不到历史任务的运行情况，所以开发时都配置历史服务器记录任务运行情况。

- 修改spark-defaults.conf.template文件名为spark-defaults.conf

  ```
  mv spark-defaults.conf.template spark-defaults.conf
  ```

- 修改spark-defalult.conf文件，配置日志存储路径

  ```
  spark.eventLog.enabled	true
  spark.eventLog.dir		hdfs://master:8020/history
  ```

  注意：需要启动hadoop集群，HDFS上的directory目录需要提前存在

  ```
  sbin/start-dfs.sh
  hadoop fs -mkdir /history
  ```

- 修改spark-env.sh文件，添加日志配置

  ```
  export SPARK_HISTORY_OPTS="-Dspark.history.ui.port=18080 -Dspark.history.fs.logDirectory=hdfs://master:8020/history -Dspark.history.retainedApplication=30"
  ```

  - 参数1含义：WEB UI访问的端口号18080
  - 参数2含义：指定历史服务器日志存储路径
  - 参数3含义：指定保存Application历史记录的个数，如果超过这个值，旧的应用程序信息将被删除，这个是内存中的应用数，而不是页面上显示的应用数。

- 分发配置文件

  ```
  scp conf slave1:/opt/apps/spark-2.4.6/
  scp conf slave2:/opt/apps/spark-2.4.6/
  ```

- 重新启动集群和历史服务

  ```
  sbin/start-all.sh
  sbin/start-history-server.sh
  ```

- 重新执行任务

  ![](http://typora-image.test.upcdn.net/images/standalone提交作业.jpg)

- 查看历史服务：http://master:18080

  ![](http://typora-image.test.upcdn.net/images/history-server.jpg)

##### 7. 配置高可用（HA）

所谓的高可用是因为当前集群中的Master节点只有一个，所以会存在单点故障问题。所以为了解决单点故障问题，需要在集群中配置多个Master节点，一旦处于活动状态的Master发生故障时，由备用Master提供服务，保证作业可以继续执行。

- 停止集群启动Zookeeper

  ```
  sbin/stop-all.sh
  zkServer.sh start
  ```

- 修改spark-env.sh文件添加如下配置

  ```shell
  # 注释如下内容：
  # SPARK_MASTER_HOST=master
  # SPARK_MASTER_PORT=7077
  
  # 添加如下内容
  SPARK_MASTER_WEBUI_PORT=8888
  export SPARK_DAEMON_JAVA_OPTS="-Dspark.deploy.recoveryMode=ZOOKEEPER -Dspark.deploy.zookeeper.url=master,slave1,slave2 -Dspark.deploy.zookeeper.dir=/spark"
  ```

- 分发配置文件

  ```
  scp conf slave1:/opt/apps/spark-2.4.6/
  scp conf slave2:/opt/apps/spark-2.4.6/
  ```

- 启动集群

  ```
  sbin/start-all.sh
  
  - 启动备用节点
  sbin/start-master.sh
  ```

  master上的Master节点

  ![](http://typora-image.test.upcdn.net/images/master.jpg)

  slave1上的备用Master节点

  ![](http://typora-image.test.upcdn.net/images/slave1.jpg)

- 提交应用到高可用集群

  ```
  bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master spark://master:7077,slave1:7077 \
  ./examples/jars/spark-examples_2.11-2.4.6.jar \
  10
  ```

#### 3. Spark的运行环境_Yarn

独立部署（Standalone）模式由Spark自身提供计算资源，无需其它框架提供资源。这种方式降低了和其它第三方资源框架的耦合性，独立性非常强。但是由于Spark本身是计算框架，所以本身提供的资源调度并不是它的强项。

##### 1. 解压缩文件

将spark-2.4.6.tgz文件上传到CentOS并解压缩，放置在指定位置。

##### 2. 修改配置文件

- 修改hadoop配置文件/opt/app/hadoop-2.7.7/etc/hadoop/yarn-site.xml，并分发

  ```xml
   <!-- 是否启动一个线程检查每个任务正使用的物理内存量，如果任务超出分配值，则直接将其杀掉，默认是true -->
  <property>
      <name>yarn.nodemanager.pmem-check-enabled</name>
      <value>false</value>
  </property>
  <!-- 是否启动一个线程检查每个任务正使用的虚拟内存量，如果任务超出分配值，则直接将其杀掉，默认是true -->
  <property>
      <name>yarn.nodemanager.vmem-check-enabled</name>
      <value>false</value>
  </property>
  ```

- 修改conf/spark-env.sh，添加JAVA_HOME和YARN_CONF_DIR配置

  ```
  export JAVA_HOME=/opt/apps/jdk1.8.0_162
  YARN_CONF_DIR=/opt/apps/hadoop-2.7.7/etc/hadoop
  ```

##### 3. 启动HDFS以及YARN集群

```
sbin/start-dfs.sh
sbin/start-yarn.sh
```

##### 4. 提交应用

```
bin/spark-submit \
--class org.apache.spark.examples.SparkPi \
--master yarn \
--deploy-mode cluster \
./examples/jars/spark-examples_2.11-2.4.6.jar \
10
```

![](http://typora-image.test.upcdn.net/images/yarn-submit.jpg)

查看http://master:8088页面，点击History，查看历史页面

![](http://typora-image.test.upcdn.net/images/yarn-history.jpg)

##### 5. 配置历史服务器

- 修改spark-defaults.conf.template文件名为spark-defaults.conf

  ```
  mv spark-defaults.conf.template spark-defaults.conf
  ```

- 修改spark-defalult.conf文件，配置日志存储路径

  ```
  spark.eventLog.enabled	true
  spark.eventLog.dir		hdfs://master:8020/history
  ```

  注意：需要启动hadoop集群，HDFS上的directory目录需要提前存在

  ```
  sbin/start-dfs.sh
  hadoop fs -mkdir /history
  ```

- 修改spark-env.sh文件，添加日志配置

  ```
  export SPARK_HISTORY_OPTS="-Dspark.history.ui.port=18080 -Dspark.history.fs.logDirectory=hdfs://master:8020/history -Dspark.history.retainedApplication=30"
  ```

  - 参数1含义：WEB UI访问的端口号18080
  - 参数2含义：指定历史服务器日志存储路径
  - 参数3含义：指定保存Application历史记录的个数，如果超过这个值，旧的应用程序信息将被删除，这个是内存中的应用数，而不是页面上显示的应用数。

- 修改spark-defaults.conf

  ```
  spark.yarn.historyServer.address=master:18080
  spark.history.ui.port=18080
  ```

- 分发配置文件

  ```
  scp conf slave1:/opt/apps/spark-2.4.6/
  scp conf slave2:/opt/apps/spark-2.4.6/
  ```

- 重新启动集群和历史服务

  ```
  sbin/start-all.sh
  sbin/start-history-server.sh
  ```

- 重新执行任务

  ```
  bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master yarn \
  --deploy-mode client \
  ./examples/jars/spark-examples_2.11-2.4.6.jar \
  10
  ```

  ![](http://typora-image.test.upcdn.net/images/yarn-client.jpg)

- 查看历史服务：http://master:18080

  ![](http://typora-image.test.upcdn.net/images/yarn-history-spark.jpg)

##### 6. Yarn Client和Yarn Cluster

这两种模式本质的区别在于AM（Application Master）进程的区别。

- Cluster：Cluster模式下，Spark Driver运行在AM中，负责向Yarn(RM)申请资源，并监督Application的运行情况。当Client提交作业后，就会关闭Client，作业会继续在Yarn上运行，这也是Cluster模式不适合交互类型作业的原因。
- Client：AM不仅仅向Yarn(RM)申请资源，之后Client会和请求的Container通信来完成任务的调度，即Client不能被关闭。

一般情况下，先在Client模式下调通任务，之后提交到Cluster上进行运行。