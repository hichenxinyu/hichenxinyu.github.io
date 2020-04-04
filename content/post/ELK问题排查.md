---
title: ELK 问题排查
date: 2019-06-03
layout: post
tags: ElasticSearch
---

### 集群 Red


为了完整的查看索引，分片，主分片和副分片情况，所以可直接使用自带 Api 查看，输入地址:

```
GET /_cat/shards?h=index,shard,prirep,state,unassigned.reason
```


  查询索引有问题的原因

```
GET /_cluster/allocation/explain
```

官网文档
[https://www.elastic.co/guide/en/elasticsearch/reference/master/cluster-allocation-explain.html](https://www.elastic.co/guide/en/elasticsearch/reference/master/cluster-allocation-explain.html)

可以查看结果，线上 ES 是12个分片，3个备份，但是每个备份都会出现一个 UNASSIGNED 的情况，我们从最后的原因 INDEX CREATED 可得知是由于创建索引的API导致未分配。

```
trackmodel-2019-01-02           8  r UNASSIGNED INDEX CREATED
trackmodel-2019-01-02           8  r STARTED 
trackmodel-2019-01-02           8  r STARTED 
trackmodel-2019-01-02           8  p STARTED 
trackmodel-2019-01-02           4  r STARTED 
trackmodel-2019-01-02           4  p STARTED 
trackmodel-2019-01-02           4  r STARTED 
trackmodel-2019-01-02           4  r UNASSIGNED INDEX CREATED
trackmodel-2019-01-02           7  r STARTED 
trackmodel-2019-01-02           7  p STARTED 
trackmodel-2019-01-02           7  r STARTED 
trackmodel-2019-01-02           7  r UNASSIGNED INDEX CREATED
.
.
.
trackmodel-2019-01-02           1  r UNASSIGNED INDEX CREATED
```

然后使用查看：
然后使用查看：

```
GET /_cat/allocation?v
```

可以从从结果中得知其实线上 ES 只有3个 node，但是由上面可知，我在创建索引时定义了3个副本，所以就是 node 数少于分片副本数导致的。

```
shards disk.indices disk.used disk.avail disk.total disk.percent host          ip            node
   204      120.4gb   135.6gb       61gb    196.7gb           68 172.16.41.127 172.16.41.127 trace-es-node-3
   204      121.9gb   139.3gb     57.4gb    196.7gb           70 172.16.41.125 172.16.41.125 trace-es-node-1
   204      120.5gb   136.9gb     59.7gb    196.7gb           69 172.16.41.126 172.16.41.126 trace-es-node-2
   204  unassinged
```


所以这个问题就很好解决了，由于本身我的索引是按每天日期新建，使用别名来进行查询，会定时删除过期索引，所以直接修改创建索引时代码，将创建副本数改为2即可解决问题。不需要更改之前出现的 UNASSIGNED 分片的问题，但是如果要更改的话可以直接使用以下命令来调整对应日期索引的副本数

```
PUT /trackmodel-2019-01-02/_settings
{  
    "number_of_replicas" : 2 
}
```


### 问题探讨

虽然这次的 UNASSIGNED 问题解决和问题原因非常简单，但是其实引起 UNASSIGNED 是有很多原因的，在 ES 开发中也经常会遇到这个问题，比如以下几种可能就都会引起 UNASSIGNED 的问题：

```
1. Shard allocation  过程中的延迟机制
2. nodes 数小于分片副本数
3. 检查是否开启 cluster.routing.allocation.enable 参数
4. 分片的历史数据丢失了
5. 磁盘不够用了
6. es 的版本问题
```

这些不同问题导致的 UNASSIGNED 都需要不同的方式去解决，所以在本文第一条命令中最后的 reason 列就相对比较重要了，是知道问题原因的初步思路来源，大概分为以下几种

```
1）INDEX_CREATED：由于创建索引的API导致未分配。
2）CLUSTER_RECOVERED ：由于完全集群恢复导致未分配。
3）INDEX_REOPENED ：由于打开open或关闭close一个索引导致未分配。
4）DANGLING_INDEX_IMPORTED ：由于导入dangling索引的结果导致未分配。
5）NEW_INDEX_RESTORED ：由于恢复到新索引导致未分配。
6）EXISTING_INDEX_RESTORED ：由于恢复到已关闭的索引导致未分配。
7）REPLICA_ADDED：由于显式添加副本分片导致未分配。
8）ALLOCATION_FAILED ：由于分片分配失败导致未分配。
9）NODE_LEFT ：由于承载该分片的节点离开集群导致未分配。
10）REINITIALIZED ：由于当分片从开始移动到初始化时导致未分配（例如，使用影子shadow副本分片）。
11）REROUTE_CANCELLED ：作为显式取消重新路由命令的结果取消分配。
12）REALLOCATED_REPLICA ：确定更好的副本位置被标定使用，导致现有的副本分配被取消，出现未分配。
```





设置单个索引在每个节点上最大分配分配数的设置是在索引模板 （index template）里设置



index.routing.allocation.total_shards_per_node":3