mysql -h cdh1 -P 9040 -uroot

```shell
docker run -it -d -v /opt/modules/.m2:/opt/modules/.m2 -v /opt/modules/maven/conf/settings.xml:/usr/share/maven/conf/settings.xml -v /opt/modules/doris/:/opt/modules/doris/ --name doris-dev apache/incubator-doris:build-env-ldb-toolchain-latest
```

```
docker run -it -d -v /opt/modules/.m2:/opt/modules/.m2 -v /opt/modules/apache-doris-1.0.0-incubating-src/:/opt/modules/apache-doris-1.0.0-incubating-src/ --name doris-1.0.0 apache/incubator-doris:build-env-for-1.0.0
```

```
docker run -it -d -v /opt/modules/.m2:/opt/modules/.m2 -v /opt/workspace/doris/:/opt/workspace/doris/ --name doris doris-dev:latest
```







```shell
docker run -it -p 8030:8030 -p 9030:9030 -d --name=doris-fe -v /opt/docker/doris/fe:/opt/doris/fe -v /opt/docker/doris/doris-meta:/opt/doris/doris-meta apache/incubator-doris:build-env-ldb-toolchain-latest
```
