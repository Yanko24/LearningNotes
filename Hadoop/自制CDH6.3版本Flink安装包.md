#### 自制CDH6.3版本Flink安装包

##### 1. 下载flink-parcel工具

```shell
git clone https://github.com/pkeropen/flink-parcel.git
```

##### 2. 修改flink版本配置

```shell
#FLINK 下载地址
FLINK_URL=https://archive.apache.org/dist/flink/flink-1.13.3/flink-1.13.3-bin-scala_2.12.tgz

#flink版本号
FLINK_VERSION=1.13.3

#扩展版本号
EXTENS_VERSION=BIN-SCALA_2.12

#操作系统版本，以centos为例
OS_VERSION=7

#CDH 小版本
CDH_MIN_FULL=5.2
CDH_MAX_FULL=6.3.3

#CDH大版本
CDH_MIN=5
CDH_MAX=6
```

##### 3. 安装maven

下载maven安装包，解压缩并配置环境变量即可。

##### 4. 打包parcel

```shell
# 进入flink-parcel目录
cd flink-parcel

# 授权
chmod a+x ./build.sh

# 打包parcel
./build.sh parcel

# 生成flink-on-yarn.jar包
./build.sh csd_on_yarn
```

##### 5. 打包parcel镜像

复制flink-parcel目录下的FLINK-1.13.3-BIN-SCALA_1.12_build和FLINK_ON_YARN-1.13.3.jar就是我们最后的parcel和jar包。



