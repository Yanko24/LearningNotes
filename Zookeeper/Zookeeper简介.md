#### Zookeeper简介

##### 1. 什么是Zookeeper？

Zookeeper是一个分布式的，开放源码的分布式应用程序协调服务，是Google的Chubby的一个开源实现，是Hadoop和HBase的重要组件。它是一个为分布式的应用提供一致性服务的软件，提供的功能包括：配置维护、域名服务、分布式同步、组服务等。

##### 2. Zookeeper安装

###### 1. 二进制安装

从[Zookeeper官网](https://zookeeper.apache.org/)下载ZK的二进制包，选择自己需要的版本进行下载。

![](images/)

下载下来是一个压缩包，解压后目录如下：

![](images/)

需要关注的是`bin`和`conf`目录，在启动之前，需要将`conf`目录下的`zoo_sample.cfg`重命名为`zoo.cfg`，因为ZK启动后默认找的配置文件就是`zoo.cfg`。之后通过`zkServer.sh start`就可以在后台启动Zookeeper进程，启动后使用`jps`命令查看可以看到`QuorumPeeMain`进程，表示ZK已经启动成功。

之后我们可以通过`zkCli.sh`连接刚刚启动的ZK服务端：

```shell
[zk: localhost:2181(CONNECTED) 0]
```

###### 2. Docker的安装

使用Docker拉取最新的镜像或者拉取指定版本的镜像：

```shell
# 默认拉取最新版本
docker pull zookeeper

# 拉取指定Tag的版本
docker pull zookeeper:3.7.0
```

在Docker中启动Zookeeper：

```shell
# 启动最新版本的zookeeper，并和主机映射2181端口对外访问。启动Zookeeper服务端的这个容器名称叫ZK
docker run -d -p 2181:2181 --name zookeeper-latest zookeeper

# 启动指定版本的zookeeper，并和主机映射2181端口对外访问。启动Zookeeper服务端的这个容器名称叫ZK
docker run -d -p 2181:2181 --name zookeeper-3.7.0 zookeeper:3.7.0
```

##### 3. Zookeeper的常见操作

新建maven项目，引入zookeeper的依赖：

```xml
<dependency>
  <groupId>org.apache.zookeeper</groupId>
  <artifactId>zookeeper</artifactId>
  <version>3.7.0</version>
</dependency>
```

Java代码操作：

```java
public class ZookeeperBaseDemo {
    private static final Logger LOG = LoggerFactory.getLogger(ZookeeperBaseDemo.class);

    public static void main(String[] args) {
        try {
            // 创建一个Zookeeper客户端对象
            ZooKeeper client = new ZooKeeper("hadoop01:2181/鸡太美", 3000, new Watcher() {
                @Override
                public void process(WatchedEvent watchedEvent) {
                    LOG.info("这是本客户端的默认回调函数！");
                }
            });

            Stat stat = client.exists("/更新视频", new Watcher() {
                @Override
                public void process(WatchedEvent watchedEvent) {
                    LOG.info("路径【/更新视频】的exists回调函数！");
                }
            });
            if (stat == null) {
                // 在父级目录下创建结点，写入数据
                // ZooDefs.Ids.OPEN_ACL_UNSAFE是一种ACL的权限，意思是不会进行权限校验
                client.create("/更新视频", "byte类型的数据".getBytes(StandardCharsets.UTF_8), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
                LOG.info("路径【/更新视频】创建成功！");
            }
            // 创建下级节点
            Stat exists = client.exists("/更新视频/20201101", true);
            if (exists == null) {
                // 创建节点
                client.create("/更新视频/20201101", "更新视频了".getBytes(StandardCharsets.UTF_8), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
                LOG.info("路径【/更新视频/20201101】创建成功！");
            }

            // 给节点设置数据，-1表示无视版本信息
            client.setData("/更新视频/20201101", "这是日期下的数据".getBytes(StandardCharsets.UTF_8), -1);
            LOG.info("路径【/更新视频/20201101】下的数据为：" + new String(client.getData("/更新视频/20201101", new Watcher() {
                @Override
                public void process(WatchedEvent watchedEvent) {
                    LOG.info("路径【/更新视频/20201101】的getData回调函数！");
                }
            }, exists)));

            // 获取子节点列表
            List<String> children = client.getChildren("/更新视频", new Watcher() {
                @Override
                public void process(WatchedEvent watchedEvent) {
                    LOG.info("路径【/更新视频】的getChildren回调函数！");
                }
            });
            LOG.info("路径【/更新视频】的子节点为：" + children);

            // 删除节点
            if (exists != null) {
                client.delete("/更新视频/20201101", -1);
            }

            // 退出时，关闭客户端连接
            client.close();
        } catch (IOException | InterruptedException | KeeperException e) {
            e.printStackTrace();
        }
    }
}
```

Tips：

Zookeeper的Java客户端在操作时，不能进行递归创建，也不能进行递归删除。创建多级路径时要保证其父节点路径时存在的；删除多级路径时，要保证其子节点路径一定不存在。

##### 4. Zookeeper的节点类型

Zookeeper中一共有7中节点类型，常见的有5中，最终两种带超时时间的节点必须开启`extendedTypesEnabled=true`配置才可以使用，否则就会报错`Unimplemented for`类似的错误。

```
PERSISTENT							// 持久节点，一旦创建成功不会被删除，除非客户端主动发起删除请求
PERSISTENT_SEQUENTIAL				// 持久顺序节点，会在用户路径后面拼接一个不会重复的自增数字后缀，其他同上
EPHEMERAL							// 临时节点，当创建该节点的客户端链接断开后自动被删除
EPHEMERAL_SEQUENTIAL				// 临时顺序节点，基本同上，也是增加一个数字后缀
CONTAINER							// 容器节点，一旦子节点被删除完就会被服务端删除
PERSISTENT_WITH_TTL					// 带过期时间的持久节点，带有超时时间的节点，如果超时时间内没有子节点被创建，就会被删除
PERSISTENT_SEQUENTIAL_WITH_TTL		// 带过期时间的持久顺序节点，基本同上，多了一个数字后缀
```

##### 5. Zookeeper节点的内存模型

整个内存对象在ZK中对应的对象其实就是DataTree：

- 其实整个Zookeeper的数据最终存储在一个哈希表（ConcurrentHashMap）中，Key是路径，而value则是对应的节点
- 节点包含了数据、子节点列表、权限、统计、版本等

##### 6. Zookeeper的订阅

Zookeeper中的判断路径是否存在、获取数据、获取子节点列表，这三种方法都可以对路径进行订阅，订阅的方式有两种：

- 传递一个`boolean`的值，如果使用该方法，回调对象就是创建Zookeeper客户端对象是的第三个参数`defaultWatcher`
- 直接在方法中传入一个`Watcher`的实现类，此实现类会作为路径之后的回调对象（推荐这个，相对于默认的回调对象，灵活度比价高）

Zookeeper在3.6.0之后支持了两种新的订阅模式：`PERSISTENT`和`PERSISTENT_RECURSIVE`（持久订阅和持久递归订阅），客户端只有通过`addWatch`才能添加这两类的订阅，删除持久的订阅也需要调用接口`removeWatches`。同一个客户端，同一个路径下只能有一个订阅类型（共三种：一次性、持久、持久递归），后面注册的类别会覆盖之前注册的类别。