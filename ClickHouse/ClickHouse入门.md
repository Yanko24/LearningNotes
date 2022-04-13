#### ClickHouse入门

##### 1. 什么是ClickHouse

ClickHouse是俄罗斯的Yonder于2016年开源的列式存储数据库，使用C++编写，主要用于联机分析处理（OLAP），能够使用SQL查询实时生成数据分析报告。

##### 2. ClickHouse的特点

###### 1. 列式存储

###### 2. DBMS的功能

###### 3. 多样化的引擎

###### 4. 高吞吐写入能力

###### 5. 数据分区与线程级并行

##### 3. ClickHouse的单机安装

###### 1. CentOS中安装ClickHouse

查看系统参数：`ulimit -a`

修改打开文件数限制：`vi /etc/security/limits.conf`，由于`/etc/security/limits.d/20-nproc.conf`可能会覆盖`limits.conf`的内容，所以同步修改其内容，如果不存在则创建。

```shell
# 用户@用户组 (软/硬)限制 nofile=打开文件数/nproc=最大打开进程数 数值
* soft nofile 65536
* hard nofile 65536
* soft nproc 131072
* hard nproc 131072
```

取消SELINUX：`/etc/selinux/config`，设置完之后需要重启，如果是生产机器无法重启，可以暂时使用`setenforce 0`临时关闭。

```shell
SELINUX=disabled
```

安装ClickHouse的依赖包：

```shell
$ sudo yum install -y libtoolw
$ sudo yum install -y *unixODBC*
```

下载对应的[rpm](https://packages.clickhouse.com/rpm/stable/)包进行安装，需要注意的是clickhouse在`20.6.3`才支持`explain`操作，所以建议安装版本`20.6.3`以上版本。

![](images/clickhouse-rpm.png)

安装完成之后默认的安装目录需要注意：

```shell
bin		==> /usr/bin
conf	==> /etc/clickhouse-server
# 以下两个可以修改
lib		==> /var/lib/clickhouse
log		==> /var/log/clickhouse-server
```

使用`rpm`命令安装，安装过程可能要输入密码，直接回车密码即为空。

```shell
$ sudo rpm -ivh *.rpm
```

使用`rpm -qa | grep clickhouse`查看是否安装成功：

![](images/clickhouse-local.png)

clickhouse的配置文件如下：

![](images/clickhouse-config.png)

修改`/etc/clickhouse-server/config.xml`中关于`listen`的配置，如果启动报错`Listen [::]:8123 failed: Poco::Exception. Code: 1000, e.code() = 0, DNS error: EAI: Address family for hostname not supported (version 21.9.7.2 (official build))`，可以将`::`修改为`0.0.0.0`并重启即可。

![](images/clickhouse-listen.png)

clickhouse命令：

```shell
# 启动clickhouse-server
$ sudo clickhouse start

# 查看clickhouse-server启动状态
$ sudo clickhouse status

# clickhouse-client客户端
$ clickhouse-client
```

###### 2. mac安装ClickHouse(M1)

直接按照官网命令，下载clickhouse，运行即可：

```shell
$ curl -O 'https://builds.clickhouse.com/master/macos-aarch64/clickhouse' && chmod a+x ./clickhouse
```

clickhouse服务启动：

```shell
# 后台以一个守护进程启动
$ ./clickhouse server --daemon

# 启动clickhouse的client
$ ./clickhouse client
```

###### 3. docker中安装ClickHouse

```shell
# 克隆docker镜像
$ docker pull yandex/clickhouse-server

# 在docker中创建容器启动clickhouse
docker run -d --name clickhouse-server --ulimit nofile=262144:262144 -p 8123:8123 -p 9000:9000 -p 9009:9009 yandex/clickhouse-server
```

##### 4. 数据类型

| 数据类型 | 对应Java类型 |       范围        |
| :------: | :----------: | :---------------: |
|   Int8   |     byte     |    [-128, 127]    |
|  Int16   |    short     |  [-32768, 32767]  |
|  Int32   |     int      | [-2^31^, 2^31^-1] |
|          |              |                   |
|          |              |                   |
|          |              |                   |
|          |              |                   |
|          |              |                   |
|          |              |                   |

