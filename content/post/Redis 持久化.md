---
title: Redis 持久化
date: 2019-07-11
layout: post
---

Redis的读写性能俱佳，但由于是内存数据库，如果没有提前备份，Redis数据是掉电即失的。
Redis提供了两种方式进行持久化：

1. RDB持久化 
1. AOF持久化


### 原理

将Redis在内存中的数据定时`dump`到磁盘上，实际操作过程是`fork`一个子进程，先将数据写入临时文件，写入成功后，再替换之前的文件，用二进制压缩存储
打印rdb文件

```
root@pa6:/var/lib/redis# od -c dump.rdb
0000000   R   E   D   I   S   0   0   0   6 376  \0  \0 003   k   e   y
0000020 301 177   # 377   ]   } 362 366 313 362   n 020
0000034
```

![image.png](https://blog-pic-1253367462.cos.ap-shanghai.myqcloud.com/1567665181186-fcf56c20-aed6-4cb4-b55b-edeff6d6c3a3.png)


#### AOF 持久化存储

AOF持久化：将Redis的操作日志以文件追加的方式写入文件，只记录写、删除操作，查询操作不会记录（类似于MySQL的Binlog日志）



![image.png](https://blog-pic-1253367462.cos.ap-shanghai.myqcloud.com/1567665214953-264e7ec5-2993-431b-9e38-6ccfd3206f57.png)



因为BGSAVE命令可以在不阻塞服务器进程的情况下执行，所以Redis允许用户通过设置服务器配置的save选项，让服务器每隔一段时间自动执行一次BGSAVE命令
用户可以通过save选项设置多个保存条件，但只要其中任意一个条件被满足，服务器就会执行BGSAVE命令 举个例子，如果我们向服务器提供以下配置
vim redis.conf

```
save 900 1
save 300 10
save 60 10000
```

那么只要满足以下三个条件中的任意一个，BGSAVE命令就会被执行 服务器在900秒之内，对数据库进行了至少1次修改
服务器在300秒之内，对数据库进行了至少10次修改
服务器在60秒之内，对数据库进行了至少10000次修改。



#### 备份脚本

```
#!/bin/bash
# 避免造成雪崩
set -e
# bak 目录
BACKUPDIR=/data/redis/backup
#PASSWD='redis密码'
#DATADIR=`/usr/local/bin/redis-cli -p 端口 -a "$PASSWD" config get dir|grep -Ev 'dir|grep'`
# 获取redis-cli 二进制文件
REDIS_CMD=$(which redis-cli)
DATADIR=`$REDIS_CMD  config get dir|grep -Ev 'dir|grep'`
DATE=`date +'%Y-%m-%d-%H-%M'`
BACKUPDIR_DATE=$BACKUPDIR_$DATE

if [ ! -d $BACKUPDIR ];then
        mkdir -p $BACKUPDIR \
        || echo "can't make dir !!!" \
        && exit 1
fi
#以日期分类
mkdir -p $BACKUPDIR/$BACKUPDIR_DATE
$REDIS_CMD  bgsave
sleep 3
cp -rf $DATADIR/* $BACKUPDIR/$BACKUPDIR_DATE
```

