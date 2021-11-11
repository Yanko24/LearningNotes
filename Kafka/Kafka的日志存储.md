#### Kafka的日志存储

kafka的消息是以topic为单位进行归类的，各个topic之间互相独立，互不影响。每个主题可以分成一个或者多个分区。每个分区各自存在一个记录消息数据的日志文件。

![](http://typora-image.test.upcdn.net/images/kafka-topic.jpg)

