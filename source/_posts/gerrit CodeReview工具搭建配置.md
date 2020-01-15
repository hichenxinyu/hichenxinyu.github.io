---
title: Gerrit CodeReview代码审核工具搭建配置
date: 2018-04-018
tags: [gitlab,gerrit,CodeReview]
---

### 一、环境准备
1. OS:centos6.8
gerrit依赖 (java&git)
yum -y install java-1.8.0
安装git 2.14.1 (低版本git同步gitlab时会出问题)
yum install -y  http://opensource.wandisco.com/centos/6/git/x86_64/wandisco-git-release-6-1.noarch.rpm
yum install -y git
yum -y install epel-release && yum -y install nginx httpd-tools

2. gerrit环境
下载：Gerrit 2.15.1  wget https://www.gerritcodereview.com/download/gerrit-2.15.1.war
<!-- more --> 

3. gerrit管理帐号(可选，使用独立账号配置gerrit)
gerrit依赖，用来管理gerrit。
sudo adduser gerrit
sudo passwd gerrit
并将gerrit加入sudo权限
sudo vi /etc/sudoers
gerrit  ALL=(ALL:ALL) ALL
备注：这里我直接使用root用户安装


### 二. 安装与配置gerrit
1. 配置gerrit
默认安装：java -jar gerrit-2.15.1.war init -d /usr/local/review(此目录最好放在空间比较大的目录)

> 配置的时候 该选择Y的时候要选择Y

Install Verified label         [y/N]? N    //如果选择默认的话 会出现没有submit按钮
Install plugin download-commands version v2.13.4 [y/N]? y  //如果不安装 会出现创建项目没有clone地址

更新配置文件：vim /usr/local/review/etc/gerrit.config

```
[gerrit]
        basePath = git
        serverId = 90b8d687-f735-4a2c-b117-0518fe571670
        canonicalWebUrl = http://192.168.2.238:8080/
[database]
        type = h2
        database = /usr/local/review/db/ReviewDB
[index]
        type = LUCENE
[auth]
        type = HTTP
[receive]
        enableSignedPush = true
[sendemail]
        enable = true
        smtpServer = smtp.exmail.qq.com
        smtpServerPort = 465  
        smtpEncryption = ssl
        smtpUser = *@*.com
        smtpPass = ***
        sslVerify = false
        from = Code Review <*@*.comm>
[container]
        user = root
        javaHome = /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.171-3.b10.el6_9.x86_64/jre
[sshd]
        listenAddress = *:29418
[httpd]
        listenUrl = http://192.168.2.238:8080/
[cache]
        directory = cache
```
2. 配置nginx反向代理
配置：vim gerrit.conf
```
server {
         listen 80;
         server_name 192.168.2.238;
         auth_basic "Welcomme to Gerrit Code Review Site!";
         auth_basic_user_file /opt/.passwd;
         location / {
            proxy_pass  http://192.168.2.238:8080;
         }
       }
```


3. 配置gerrit账户密码 创建认证用户(通过httpd-tools工具)
touch /opt/.passwd
htpasswd -c -b /opt/.passwd admin 123456(管理员)

4. 启动gerrit&启动nginx
sudo /usr/local/review/bin/gerrit.sh restart
service nginx start

5. 访问gerrit 管理界面 http://xxx.xxxxx.com/
第一次访问，需要输入第3步设置的admin及密码，该账户将作为gerrit管理员账户。

### 三、如何使用gerrit
前提：需要git使用端 / gerrit服务端配合使用。
1. 添加项目(gerrit 服务端)
1.1 使用gerrit添加新项目：（适用于开启新项目并使用gerrit）
使用gerrit管理界面创建项目

1.2使用gerrit添加已有项目：（适用于已有项目下移植到gerrit中）
ssh -p 3389 gerrit1@公网IP gerrit create-project --name exist-project #建议采用管理界面添加
或者使用gerrit管理界面
然后将已有项目与gerrit上建立的exist-project关联，即将已有代码库代码push到gerrit中进行管理。
cd /usr/local/review/git/exist-project
git push ssh://gerrit1@公网IP:3389/exist-project *:*

2. 生成sshkey(git使用端)
在开发账户中生成sshkey，用作与gerrit服务器连接。
ssh-keygen -t rsa #生成sshkey

cat ~/.ssh/id_rsa.pub #查看sshkey

3. 添加sshkey到gerrit服务器(gerrit 服务端)
此步骤与git流程类似，即将/root/.ssh/id_rsa.pub内容添加到gerri和gitlab(后面gerrit和gitlab直接同步)

4. 拉取代码＆配置git hooks(git client端)
验证sshkey是否配置成功：ssh -p 29418 admin@192.168.199.222

拉取代码url：选择 clone with commit-msg hook  //会直接配置Change-ID gerrit流程必备

修改代码并提交，推送时与原有git流程不一致，采用 git push origin HEAD:refs/for/master

### 四.使用gerrit website完成code review
当完成push后，可在gerrit管理界面看到当前提交code review的change。
查看需要code review的提交：
查看某次提交的详细信息（审核者+2可通过本次提交，提交者可通过Abandon本次提交）：
如果审核者+2通过后，可提交该次commit.

##### 配置gerrit和gitlab同步
gerrit和gitlab同步需要配置两个地方 (前提示已将sshkey 添加至gerrit和gitlab)

配置 /root/.ssh/config
````
Host 192.168.2.244
    User root
    IdentityFile ~/.ssh/id_rsa
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
    PreferredAuthentications publickey
````

配置replation插件
/usr/local/review/etc/replication.config   //以实际gerrit安装位置为准

````
[remote "192.168.2.244"]
         url = git@192.168.2.244:root/${name}.git
         push = +refs/heads/*:refs/heads/*
         push = +refs/tags/*:refs/tags/*
         push = +refs/changes/*:refs/changes/*
         timtout = 30
         threads = 3
````

### 五.gerrit注意事项
* 需要为每个使用者分配gerrit账号，不要都使用admin账号，因为admin账号可直接push master
* push代码时需要使用
git push origin HEAD:refs/for/master(branch),gerrit默认关闭非admin账号的push direct权限
* push代码时需要commit email与gerrit logoutaccount email一致，否则无法push成功，可选择关闭email notify，并开启forge user权限，或者通过修改gerrit数据库account email信息
* git配置的email一定要和gerrit里注册的email一致，否者push会出错
* gerrit数据库与gitlab同步，需要安装replication插件，并开启该功能
