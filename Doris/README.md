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

动态修改参数：

```
curl -X POST http://{be_ip}:{be_http_port}/api/update_config?{key}={value}'

curl -X POST http://{be_ip}:{be_http_port}/api/update_config?{key}={value}&persist=true'
```

```
max_segment_num_per_rowset
streaming_load_max_mb

exec_mem_limit 10737418240
```

```
fe配置：
max_broker_concurrency=BE个数
max_bytes_per_broker_scanner>=当前导入任务单个 BE 处理的数据量
```

