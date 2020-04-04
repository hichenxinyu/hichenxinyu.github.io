---
title: Kafka & ZooKeeper 集群搭建
date: 2019-07-11
layout: post
tags: Kafka
---

####  环境准备
三台机器 CentOS 7.4
10.0.19.218
10.0.19.216
10.0.19.142


kafka 集群搭建准备:

- Java环境
- ZooKeeper 集群搭建

#### Zookeeper 集群搭建

```
server.1=10.0.19.218:2888:3888
server.2=10.0.19.216:2888:3888
server.3=10.0.19.142:2888:3888
```


```
echo "1" >/tmp/zookeeper/myid ##生成ID，这里需要注意， myid对应的zoo.cfg的server.ID，比如第二台zookeeper主机对应的myid应该是2
```


启动ZooKeeper

```
cd /usr/local/zookeeper-3.4.9/bin
./zkServer.sh start
```


停止Zookeeper

```
cd /usr/local/zookeeper-3.4.9/bin
./zkServer.sh stop
```


附美味测试环境Zookeeper 集群配置

```
[root@rdpops_server-146-199 conf]# cat zoo.cfg
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just
# example sakes.
dataDir=/var/zookeeper/
dataLogDir=/var/zookeeper/datalog/
# the port at which the clients will connect
clientPort=2181
# the maximum number of client connections.
# increase this if you need to handle more clients
maxClientCnxns=0
#
# Be sure to read the maintenance section of the
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
autopurge.snapRetainCount=30
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1
server.1=10.0.146.126:2888:3888
server.2=10.0.146.150:2888:3888
server.3=10.0.146.199:2888:3888
```

---


#### Kafka  集群搭建

下载安装包

```
wget http://apache.claz.org/kafka/2.2.0/kafka_2.11-2.2.0.tgz
tar -xzf kafka_2.11-2.2.0.tgz
cd kafka_2.11-2.2.0
```

#### 启动服务

##### kafka 依赖 zookeeper

##### 启动 Kafka server:

```
./kafka-server-start.sh -daemon ../config/server.properties 
```



#### Kafka Web管理工具-Kafka-manager



#### 消息生产 测试

```
./kafka-console-producer.sh --broker-list localhost:9092 --topic demo
./kafka-console-consumer.sh --bootstrap-server localhost:9092 --from-beginning --topic demo
```

