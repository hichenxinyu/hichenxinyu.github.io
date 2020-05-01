---
title: Zabbix4.4安装
date: 2017-11-28
layout: post
---


### Zabbix介绍

> Zabbix 是一个企业级的分布式开源监控方案。
> zabbix server ，zabbix agent> 与可选组件zabbix proxy。

**API 功能 ：** 应用api功能，可以方便的和其他系统结合，包括手机客户端的使用。

更多功能请查看[http://www.zabbix.com/documentation.php](http://www.zabbix.com/documentation.php)



### 安装zabbix环境及准备工作

zabbix 依赖 LAMP （LNMP 也可以）

#####  安装zabbix

- （单机）--> LAMP
- （架构）--> LAP + MYSQL（生产环境，建议数据库与 Zabbix-Server 分开部署）
- 服务端端口：10051
- 客户端端口：10050

---

1，安装Zabbix需要的硬件环境及软件版本，我这里在官网上查了一下，你可以根据自己的环境和要求来选择：<br />下表是几个硬件配置的示例:

| 名称       | 平台                      | **CPU/**内存       | 数据库                                | 监控主机数量 |
| ---------- | ------------------------- | ------------------ | ------------------------------------- | ------------ |
| **小型**   | CentOS                    | 虚拟应用           | MySQL   InnoDB                        | 100          |
| **中型**   | CentOS                    | 2 CPU   cores/2GB  | MySQL   InnoDB                        | 500          |
| **大型**   | RedHat   Enterprise Linux | 4 CPU   cores/8GB  | RAID10   MySQL InnoDB or PostgreSQL   | >1000        |
| **巨大型** | RedHat   Enterprise Linux | 8 CPU   cores/16GB | 快速RAID10 MySQL InnoDB or PostgreSQL | >10000       |


具体的配置极其依赖于Active Item数量和轮询频率。如需要进行大规模部署，强烈建议将数据库进行独立部署。

---

2，接下来我说一下我实验环境

| 操作系统  | 主机IP        | 主机名称 | 安装软件      | 安装zabbix版本      | MySQL版本   |
| --------- | ------------- | -------- | ------------- | ------------------- | ----------- |
| Centos7.3 | 192.68.0.20   | zabbix   | Zabbix-server | Zabbix 3.4.10       | MySQL5.7.22 |
| centos6.5 | 192.168.0.157 | Test02   | zabbix-agent  | zabbix-agent-3.4.10 | /           |




### 安装zabbix

3.1）在监控主机上需要预先安装yum 源，下面正式开始安装；

```
rpm -Uvh https://repo.zabbix.com/zabbix/4.4/rhel/6/x86_64/zabbix-release-4.4-1.el6.noarch.rpm
yum clean all
```

3.2）安装Zabbix-server包和zabbix-agent包

```
yum -y install zabbix-server-mysql zabbix-web-mysql zabbix-agent
```

3.3）下载安装mysql源

```
rpm -ivh https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm
```

3.4)查看当前可用的Mysql安装源

```
[root@zabbix ~]# yum repolist enabled | grep "mysql.*-community.*"
mysql-connectors-community/x86_64 MySQL Connectors Community                  51
mysql-tools-community/x86_64      MySQL Tools Community                       63
mysql57-community/x86_64          MySQL 5.7 Community Server                 267
```



### MySQL 安装&配置

```
yum -y install mysql-community-server
```

3.6）启动mysql服务并设置开机启动

```
[root@zabbix ~]#systemctl start mysqld
[root@zabbix ~]#systemctl enable mysqld
```

3.7)进入MySQL并修改密码

```
[root@zabbix ~]# cat /var/log/mysqld.log | grep password
或者：/usr/bin/mysqladmin -u root password 'new-password'

[root@zabbix ~]# mysql -uroot -pnew-password
mysql> SET PASSWORD = PASSWORD ('Pass123!');
如果想用简单的密码必须先改一个变量；
mysql> set global validate_password_policy=0;
mysql> SET PASSWORD = PASSWORD ('12345678');
不然你改密码会不通过，会有密码复杂度要求。
```

3.8）创建数据库和zabbix用户并授权

```
mysql> create database zabbix character set utf8 collate utf8_bin;
Query OK, 1 row affected (10.03 sec)
mysql> grant all privileges on zabbix.* to zabbix@localhost identified by 'Pass123!';
Query OK, 0 rows affected, 1 warning (0.00 sec)
```

3.9)导入初始架构（Schema）和数据

```
[root@zabbix ~]#cd zabbix-server-mysql-4.4.1/
[root@zabbix  zabbix-server-mysql-3.4.10 ~]#zcat create.sql.gz | mysql -uzabbix -pPass123! -D zabbix
mysql: [Warning] Using a password on the command line interface can be insecure.
```

3.10)然后进入mysql查看这些内容是否导入进去

```
mysql> show tables from zabbix;
```

![image.png](https://blog-pic-1253367462.cos.ap-shanghai.myqcloud.com/1574069473892-5a56476b-3d7f-46f5-b559-02bb58924e09.png)







```
mysql> select count(*) tables,table_schema from information_schema.tables where table_schema ="zabbix";
```



---

4.修改配置文件，给服务授权、启动Zabbix Server服务<br />4.1）修改配置文件

```
备注：记得先备份 cp /etc/zabbix/zabbix_server.conf  /etc/zabbix/zabbix_server.conf.bak 
[root@zabbix ~]#vim  /etc/zabbix/zabbix_server.conf
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=Pass123!
```

4.2）给服务授权

```
[root@zabbix ~]#chown -R zabbix:zabbix /etc/zabbix/
[root@zabbix ~]#chmod -R 755 /etc/zabbix/
```

##### 启动Zabbix Server服务

```
[root@zabbix ~]#systemctl start  zabbix-server
[root@zabbix ~]#systemctl enable zabbix-server
```

备注：这里会有一个坑，就是在启动zabbix服务会失败，Job for zabbix-server.service failed. See 'systemctl status zabbix-server.service' and 'journalctl -xn' for details.查了一下原因是gnutls-3.3的高版本问题，解决办法是;1,先卸载这个高版本的gnutls-3.3,命令：rpm -e gnutls-3.3.24-1.el7.x86_64 --nodeps2，然后去网上下载一个gnutls-3.1的版本，然后使用命令rpm -Uvh --force gnutls-3.1.18-8.el7.x86_64.rpm

---


<a name="13601554"></a>

##### 编辑Zabbix前端的PHP配置

- Zabbix前端的Apache配置文件位于 /etc/httpd/conf.d/zabbix.conf 。一些PHP设置已经完成了配置。


```
#CentOS6 中获取zabbix apache 配置路径命令
rpm -ql zabbix-web | grep example.conf
```

```
[root@zabbix ~]# vim /etc/httpd/conf.d/zabbix.conf
找到<IfModule mod_php5.c>标签下面
添加一条
php_value date.timezone Asia/Shanghai
```

- 启动apache服务，并设置开机自启

```
[root@zabbix ~]#systemctl start httpd
[root@zabbix ~]#systemctl enable  httpd
```

---


<a name="2bbf3759"></a>

#### Zabbix Web 页面配置

1，访问ip：[http://IP/zabbix/index.php](http://192.168.0.20/zabbix/index.php)<br />

中间省略一部分-----------------------------直接到登录界面了。<br />

![](https://blog-pic-1253367462.cos.ap-shanghai.myqcloud.com/image-20200324220001544.png)默认的用户名是：Admin 密码：zabbix<br />



#### Zabbix Agent

- 添加一台Linux客户端机器（ip:192.168.0.157）

访问zabbix官网：[https://www.zabbix.com/download?zabbix=3.4&os_distribution=centos&os_version=6&db=MySQL](https://www.zabbix.com/download?zabbix=3.4&os_distribution=centos&os_version=6&db=MySQL)

- 添加Yum源:

```
#Centos6
rpm -ivh  http://repo.zabbix.com/zabbix/3.4/rhel/6/x86_64/zabbix-release-3.4-1.el6.noarch.rpm
#Centos7 
rpm -ivh http://repo.zabbix.com/zabbix/3.4/rhel/7/x86_64/zabbix-agent-3.4.1-1.el7.x86_64.rpm
```

- 安装客户端agent软件

```
yum -y install zabbix-agent
```

- 修改agent配置文件

```
grep -v '^$' /etc/zabbix/zabbix_agentd.conf |grep -v '^#'

PidFile=/var/run/zabbix/zabbix_agentd.pid
LogFile=/var/log/zabbix/zabbix_agentd.log
LogFileSize=0
Server=192.168.2.82
ServerActive=192.168.2.82:10050
Hostname=Test02
Include=/etc/zabbix/zabbix_agentd.d/*.conf
```

```
# 给配置文件授权
chmod 775 /etc/zabbix/zabbix_agentd.conf
# 启动agent服务并查看服务启动成功没有
/etc/init.d/zabbix-agent start 
netstat -lntup |grep zabbix_agent
```

- Web 页面添加第一台主机

  - 配置--主机---创建主机

    ![image-20200324215908603](https://blog-pic-1253367462.cos.ap-shanghai.myqcloud.com/image-20200324215908603.png)

---


- 添加主机详细信息

![image-20200324215844994](https://blog-pic-1253367462.cos.ap-shanghai.myqcloud.com/image-20200324215844994.png)

##### 添加主机模板信息

![image-20200324215828877](https://blog-pic-1253367462.cos.ap-shanghai.myqcloud.com/image-20200324215828877.png)


这样一台客户端Linux基本添加完成，过几分钟就能开到Zabbix图标变绿证明添加成功了。

