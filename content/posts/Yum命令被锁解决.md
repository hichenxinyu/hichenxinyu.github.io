---
title: Yum 命令被锁
date: 2018-02-15
layout: post
---

```
root@centos001 ~]# yum clean all      //执行一个yum命令
已加载插件：fastestmirror
Repodata is over 2 weeks old. Install yum-cron? Or run: yum makecache fast
/var/run/yum.pid 已被锁定，PID 为 2308 的另一个程序正在运行。  //可以看到文件被锁定

```
<!-- more --> 


#### 此处发现命令被锁，无法使用新的yum命令

解决方法

1. 先用CTRL+Z暂停命令
2. 输入命令：
```
rm -f /var/run/yum.pid    //删除这个文件
```
3 在执行yum命令就ok了
```
[root@centos001 ~]# yum clean all
已加载插件：fastestmirror
正在清理软件源： base extras updates
Cleaning up everything
```
