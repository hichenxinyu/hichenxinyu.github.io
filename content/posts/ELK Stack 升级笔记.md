---
title: ELK Stack 升级
date: 2019-06-11
layout: post
---

### 升级方案

比较常见的 ELK Stack 升级方式有一下 3 种

- 数据迁移升级：新建一套高版本 ES 集群，然后把老的 ES数据导入 
- 滚动升级，5.0.0->5.6,5.6->6.8.2
- 集群重启升级，5.0.0->6.8.2


| 升级方式 | 优点 | 缺点 |
| --- | --- | --- |
| 数据迁移&升级 | 可以跨大版本升级 | 升级过程中，旧集群写操作停止
ES 版本之间有差异比较大，可能会有一些坑
迁移时间较长 |
| 滚动升级 | 升级过程中服务不中断 | 对ELK 源版本与目标版本有要求
 |
| 集群重启升级 | 可以跨大版本升级
迁移时间比较短 | ES 服务中断
 |


公司生产环境与测试环境的 ELK 技术栈版本是 5.0.0
考虑先滚动升级到 5.6.16  然后确认没问题后，再从 5.6.16 滚动升级到 6.8.2
升级一般很快，但是最麻烦的不是升级，而是保证 ES 中的索引正常。
一般不建议跨大版本升级，而是小版本迭代。

---

### 数据备份

备份方式：Elasticsearch snapshot
IDC 测试环境可以考虑快照备份到本地 HDFS，磁盘
生产环境 ES 可以快照备份到阿里云 OSS

> 需要安装对应的 ES 插件

本次测试环境 ELK 集群升级，不备份

#### 备份你的 Kibana 数据 （生产环境中非常重要）

你的 Kibana 中的 index pattern 等数据都是存在 .Kibana 这个索引中的 
具体备份&升级可以参考官方文档
[https://www.elastic.co/guide/en/kibana/6.3/migrating-6.0-index.html](https://www.elastic.co/guide/en/kibana/6.3/migrating-6.0-index.html)
[https://www.elastic.co/guide/en/elasticsearch/reference/5.6/modules-snapshots.html](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/modules-snapshots.html)

---

### 升级前准备

升级Elasticsearch前，先停止Logstash（hangout）服务，不再消费 kafka 中消息（Filebeat服务无需关闭）

#### 禁用集群的分片自动分配功能

```
PUT _cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.enable": "none"
  }
}
```

启动集群分片自动分配

```
PUT _cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.enable": null
  }
}
```


### 下面这个待确认

```
PUT _cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.enable": "primaries"
  }
}
```

#### 同步数据（强制刷新，尽可能将缓存中的数据，写入磁盘）

通过 synced flush，可以在索引上放置一个 sync ID，这样可以提高这些分片Recovery的时间

```
POST _flush/synced
```

---

### 升级 ES

注意，这里使用rpm安装时，不使用-Uvh选项升级，而是先卸载旧的版本，再使用-ivh安装。另外，旧的配置文件不建议继续使用，而是将所配置内容替换到新版的配置文件中，除了个别配置外大部分仍然兼容。

Elasticsearch节点，依次执行

```
#停止当前ES 节点
service elasticsearch stop
#卸载所有 ES 插件  进入/usr/share/elasticsearch目录
#elasticsearch-plugin list
#elasticsearch-plugin remove
# 卸载旧版本注意，卸载后，旧的配置文件会以rpmsave后缀保留
rpm -e elasticsearch
```

##### 安装新版本

```
rpm -ivh /usr/local/elasticsearch-5.6.16.rpm
```

##### 替换配置文件

```
cat /etc/elasticsearch/elasticsearch.yml.rpmsave > /etc/elasticsearch/elasticsearch.yml
cat /etc/elasticsearch/jvm.options.rpmsave > /etc/elasticsearch/jvm.options
```

##### 启动Elasticsearch

```
service elasticsearch start
```

##### 开启集群的分片自动分配功能

**

```
PUT _cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.enable": null
  }
}
```

##### 等待集群恢复完毕

```
GET _cat/health
```

查看集群状态，Green 则恢复正常，升级成功

---

### 验证方式

API 验证集群状态
Cerebro 中观察分片分配情况


### 升级ES过程中可能会遇到的问题：

##### 启动失败 

```
[2019-09-16T20:04:45,116][ERROR][o.e.b.Bootstrap          ] [node-1] node validation exception
[1] bootstrap checks failed
[1]: system call filters failed to install; check the logs and fix your configuration or disable system call filters at your own risk
```

##### /etc/elasticsearch/elasticsearch.yml  添加一行配置

```
bootstrap.system_call_filter: false
```

> 官方文档见：
> [https://www.elastic.co/guide/en/elasticsearch/reference/5.6/system-call-filter-check.html](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/system-call-filter-check.html)



---

### 升级失败回滚降级

```
#停止当前ES 节点
service elasticsearch stop
#卸载所有 ES 插件  进入/usr/share/elasticsearch目录
#elasticsearch-plugin list
#elasticsearch-plugin remove
# 卸载当前版本 ES
rpm -e elasticsearch
#重新安装 5.0.0 版本 ES
rpm install /usr/local/elasticsearch-5.0.0.rpm
#拷贝之前备份的配置文件
#启动 ES
sevice elasticsearch start
```

---


### 升级Kibana

> 重点在升级配置
> 手动（ansible 脚本）将旧的配置文件中的配置，替换到新版配置文件中的对应条目中，注意兼容


```
#停止Kibana
service kibana stop
#卸载旧版本
rpm -e kibana
#安装新版本 Kibana
rpm -ivh /usr/local/kibana-5.6.16-x86_64.rpm
```

#### 升级配置

```
#启动Kibana
service kibana start 
```

##### 固化脚本（见 Ansible 脚本）

```
#停止Kibana
service kibana stop
#卸载旧版本
rpm -e kibana
#安装新版本
rpm -ivh XX.rpm
#升级配置

#启动Kibana
service kibana start
```


迁移 Kibana 数据    
Kibana 数据存储在   Elasticsearch   .Kibana 索引中
比如下图中  生产环境 ELK 日志系统  有 4 个 kibana 实例，在 ES 集群中对应这有 4 个 .kibana 索引 
![image.png](https://cdn.nlark.com/yuque/0/2019/png/330515/1576133404676-29e785e5-1c83-4b9f-a688-7f46e9563c59.png#align=left&display=inline&height=530&name=image.png&originHeight=1060&originWidth=2470&size=205099&status=done&style=none&width=1235)   

---


### 升级 Filebeat

本次不升级 Filebeat







---

### 升级的一些坑

Kibana 偶现无法访问
升级 Kibana 的一个小坑，从 6.3 之后
server.rewriteBasePath 由默认的 false 改为了 true 这个值我们没配置过
所以在 Kibana 前面有反向代理的情况下 ，会有重写代理请求的可能
也就是我们访问 Kibana 没响应，但是Kibana服务其实还在

```
# This setting cannot end in a slash.
#server.basePath: ""
server.basePath: "/anchang"
# Specifies whether Kibana should rewrite requests that are prefixed with
# `server.basePath` or require that they are rewritten by your reverse proxy.
# This setting was effectively always `false` before Kibana 6.3 and will
# default to `true` starting in Kibana 7.0.
server.rewriteBasePath: false
```

