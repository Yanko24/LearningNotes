### 1. 扩容BE

将doris-1.1.0目录下的`be`安装包分发到新增的Doris节点，并修改相关配置。

修改完成后在FE执行添加BE操作：

```sql
alter system add backend "be_host:heartbeat-service_port";
```

### 2. 扩容Broker

#### 2.1 分发TDH-Client

因为Broker需要和TDH交互，所以将`tdh-client.tar`分发到新增的Doris节点，并修改其目录下所有的`core-site.xml`中的如下参数：

```xml
<property>
    <name>hadoop.security.authentication.oauth2.enabled</name>
    <value>false</value>
</property>
```

注：需要修改的目录如下，安装完成后需要source目录下的init.sh，建议最好写入系统环境变量，自动加载。

```
{root}/TDH-Client/conf/hdfs1/core-site.xml
{root}/TDH-Client/conf/yarn1/core-site.xml
```

#### 2.2 调整相关Kerberos配置



将doris-1.1.0目录下的`apache_hdfs_broker`安装包分发到新增的Doris节点，同时将