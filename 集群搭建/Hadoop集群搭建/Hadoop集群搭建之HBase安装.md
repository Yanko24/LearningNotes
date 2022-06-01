### Hadoop集群搭建之HBase安装

#### 1. 准备工作

准备好已经安装了Hadoop的集群服务器之后，下载HBase的安装包并上传至其中一台服务器中，[下载地址](https://downloads.apache.org/hbase/)。

由于hadoop-3.3.1（HA）和hbase-2.4.8安装存在部分问题，编译hbase源码安装，需要下载hbase-2.4.8-src.tar.gz

#### 2. HBase独立模式安装

##### 2.1 安装目录规划

```
统一安装路径：/opt/modules
统一软件存放路径：/opt/software
```

##### 2.2 上传压缩包

```
1. 将压缩包上传到[/opt/software]目录下，解压到[/opt/modules]目录下
2. 修改[/home/hadoop/.bash_profile]文件，增加以下内容：
	HBASE_HOME=/opt/modules/hbase
	PATH=$HBASE_HOME/bin:$PATH
	export HBASE_HOME PATH
3. 使用[source ~/.bash_profile]使其生效
```

##### 2.3 HBase配置

配置文件目录：【/opt/modules/hbase/conf/】

###### 2.3.1 hbase-env.sh

```
JAVA_HOME的路径为[/opt/modules/jdk]
HBASE_MANAGES_ZK=true
```

###### 2.3.2 hbase-site.xml

```xml
<configuration>
    <property>
        <name>hbase.rootdir</name>
        <value>file:///opt/modules/hbase/data</value>
    </property>
    <property>
        <name>hbase.zookeeper.property.dataDir</name>
        <value>/opt/modules/hbase/zookeeper</value>
    </property>
</configuration>
```

##### 2.5 启动HBase守护进程

```shell
# 启动HBase
start-hbase.sh
```

##### 2.6 测试本地安装

``` shell
# 连接HBase
hbase shell
```

#### 3. HBase源码编译

```shell
wget https://downloads.apache.org/hbase/2.4.8/hbase-2.4.8-src.tar.gz

# 解压后进入hbase-2.4.8目录
tar -zvxf hbase-2.4.8-src.tar.gz
cd hbase-2.4.8

# 提前安装好maven，编译源码
mvn clean package -DskipTest assembly:single -Dhadoop.profile=3.0 -Dhadoop-three.version-3.3.1

# 编译成功后，在hbase-assembly/target目录下可以看到hbase-2.4.8-bin.tar.gz
```

#### 4. HBase伪分布式安装

HBase配置：配置文件目录：【/opt/modules/hbase/conf/】，需要将hadoop的hdfs-site.xml和core-site.xml复制到hbase的conf目录。

##### 4.1 hbase-env.sh

```
JAVA_HOME的路径为[/opt/modules/jdk]
HBASE_MANAGES_ZK=false
```

##### 4.2 hbase-site.xml

```xml
<configuration>
    <property>
         <name>hbase.rootdir</name>
         <value>hdfs://supercluster/hbase</value>
    </property>
    <property>
            <name>hbase.zookeeper.quorum</name>
            <value>hadoop02:2181,hadoop03:2181,hadoop04:2181</value>
    </property>
    <property>
         <name>hbase.cluster.distributed</name>
         <value>true</value>
    </property>
    <property>
        <name>hbase.unsafe.stream.capability.enforce</name>
        <value>true</value>
    </property>
</configuration>
```

##### 4.3 启动HBase服务

```shell
# 首先启动zookeeper服务
zkServer.sh start

# 启动HMaster进程(可以同时启动backup-master
# 2 3 5 是偏移量，启动在不同的端口
local-master-backup.sh start 2 3 5
# 停止HMaster
local-master-backup.sh stop 2 3 5

# 启动RegionServer
local-regionservers.sh start 2 3 5
# 停止RegionServer
local-regionservers.sh stop 2 3 5
```

#### 5. HBase全分布式安装

##### 5.1 HBase进程规划

| 主机名称 |     IP地址     |  用户  |             HBase             |   Zookeeper    |
| :------: | :------------: | :----: | :---------------------------: | :------------: |
| hadoop01 | 192.168.21.210 | hadoop |    HMaster，HRegionServer     |                |
| hadoop02 | 192.168.21.211 | hadoop | Backer HMaster，HRegionServer | QuorumPeerMain |
| hadoop03 | 192.168.21.212 | hadoop |         HRegionServer         | QuorumPeerMain |
| hadoop04 | 192.168.21.214 | hadoop |         HRegionServer         | QuorumPeerMain |

##### 5.2 HBase配置

配置文件目录：【/opt/modules/hbase/conf/】，需要将hadoop的hdfs-site.xml和core-site.xml复制到hbase的conf目录。

###### 5.2.1 hbase-env.sh

```
JAVA_HOME的路径为[/opt/modules/jdk]
HBASE_MANAGES_ZK=false
```

###### 5.2.2 hbase-site.xml

```xml
<configuration>
    <property>
	<name>hbase.rootdir</name>
	<value>hdfs://supercluster/hbase</value>
    </property>
    <property>
	<name>hbase.zookeeper.quorum</name>
	<value>hadoop02:2181,hadoop03:2181,hadoop04:2181</value>
    </property>
    <property>
        <name>hbase.cluster.distributed</name>
        <value>true</value>
    </property>
    <property>
        <name>hbase.unsafe.stream.capability.enforce</name>
        <value>true</value>
    </property>
</configuration>
```

###### 5.2.3 配置regionserver的节点信息

```shell
vi /opt/modules/hbase/conf/regionservers
# 添加如下内容
hadoop02
hadoop03
hadoop04
```

###### 5.2.4 配置backup-hmaster，需要创建backup-masters文件

```shell
echo "hadoop02" >> /opt/modules/hbase/conf/backup-masters
```

###### 5.2.5 将HBase安装包发送至其他节点，并配置环境变量

```shell
scp /opt/modules/hbase/ hostname:/opt/modules/
```

##### 5.3 启动集群并测试

```shell
start-hbase.sh
```

##### 5.4 查看WebUI页面

```
http://ip:16010
```

#### 6. 其它问题

##### 6.1 Hadoop是HA的情况

```
将Hadoop目录下的hdfs-site.xml和core-site.xml复制到HBase目录的conf目录下，重新启动
```

