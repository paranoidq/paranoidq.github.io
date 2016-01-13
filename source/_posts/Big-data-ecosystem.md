title: 大数据生态圈技术总结（持续补充）
tags: [大数据, todo]
categories: 大数据
---

### 分布式文件系统
1. 磁盘
 - HDFS
 - S3
 - Ceph
 - NFS
 - Gluster FS

2. 内存
 - Tachyon
 - Spark

### 分布式数据库
1. 磁盘
 - Cassandra
 - HBase
 - MongoDB

2. 内存 
 - Redis
 - Memcached


### 分布式计算
1. 批处理
 - Hadoop MapReduce
 - Spark(支持迭代)
 - Flink(支持迭代)

2. 流式计算
 - Storm
 - Samza
 - Spark Streaming
 - Flink

3. 即席查询(ad-hoc)
 - Hive
 - SparkSQL
 - Presto(Facebook)
 - Impala
 - Drill(Google Dremel的开源实现)


### 资源调度与管理
 - ZooKeeper
 - YARN
 - Mesos

### 分布式消息系统
 - StormMQ
 - RabbitMQ
 - ZeroMQ
 - Apache ActiveMQ
 - Jafka(LinkedIn)
 - Kafka(LinkedIn)

### RPC框架
 - Apache Avro
 - Thrift(Facebook)
 - Kyro

### 集群监控
 - Zabbix
 - Ganglia
 - Nagios
 - Ambari()

### 数据收集
 - Flume
 - Scribe(Facebook)
 - Logstash
 - Kafka

### 图计算框架
 - Spark Graphx
 - PowerGraph
 - Giraph
 - Neo4j

### 大规模机器学习
 - Spark MLlib
 - Mahout
 - PredictionIO

### 搜索引擎
 - Lucene
 - Solr
 - ElasticSearch
 - Sphinx
 - SenseiDB

### IaaS
 - OpenStack
 - Docker 
 - Kubernetes(容器调度管理)

### 基础结构
 - LevelDB
 - SSTable(BigTable基础)
 - RecordIO(文件格式)
 - Flat Buffer(Google, 高效、跨平台的序列化库)
 - ProtocolBuffers(Google, 数据描述语言，类似于XML能够将结构化数据序列化，可用于数据存储、通信协议等方面)
 - Consistent Hashing
 - Netty(提供异步的、事件驱动的网络应用程序框架和工具，用以快速开发高性能、高可靠性的网络服务器和客户端程序)
 - BloomFilter


### 参考
[http://www.36dsj.com/archives/25042](http://www.36dsj.com/archives/25042)
[http://www.csdn.net/article/2015-09-11/2825674](http://www.csdn.net/article/2015-09-11/2825674)


