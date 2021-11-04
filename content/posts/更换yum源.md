---
title: Centos更换yum源
date: 2018-02-02 11:12:48
layout: post
categories:
  - Life
tags:
---

#### 第一步：备份你的原镜像文件，以免出错后可以恢复。

````
cp -a /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak

````

#### 第二步：下载新的CentOS-Base.repo 到/etc/yum.repos.d/
````
CentOS 7
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.huaweicloud.com/repository/conf/CentOS-7-anon.repo
curl -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
````
<!-- more --> 

 

#### 第三步：运行yum makecache生成缓存
````
yum clean all
yum makecache
````
