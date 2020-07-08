---
title: Jenkins 使用总结
date: 2020-04-30
layout: post
---

#### Jenkins优势

- 编译服务和代码仓库分离，耦合度低
- 插件丰富，支持语言众多。
- 有统一的web管理界面

#### Jenkins劣势

- 插件以及自身安装较为复杂
- 对docker插件支持不是很友好
- 体量较大，不是很适合小型团队



#### Jenkins 安装：

Jenkins 安装异常简单，按照官网文档  几分钟就搞定了

先安装个jdk ，然后直接rpm安装

```
rpm -ivh https://mirrors.cloud.tencent.com/jenkins/redhat-stable/jenkins-2.222.1-1.1.noarch.rpm
```

Docker 部署

```
  docker run -u root --rm -d -p 8080:8080 -p 50000:50000  -v jenkins-data:/var/jenkins_home  -v /var/run/docker.sock:/var/run/docker.sock jenkinsci/blueocean
```

部署到K8s 中 保证高可用 (持久化存储)

```
#可以使用官方镜像
jenkins/jenkins:lts
```



### Jenkins 主从

Jenkins本身是单体的，即只能有一个Jenkins Master，但是可以设置多个Agent （Slave）节点

Slave 节点（Agent 节点）无需安装Jenkins，只要能保证

- master 节点机器 能通过ssh登录slave 节点机器

- Java环境（路径必须是 /usr/local/bin/java）

  

同时也利用 Kubernetes 来动态运行 Jenkins 的 Slave 节点，可以和好的来解决传统的 Jenkins Slave 浪费大量资源的缺点。



### 创建 Slave 节点

- Jenkins --> 系统管理 --> 管理节点 --> 新建节点

- 填写节点信息

  ​	  Remote root directory 表示节点的工作目录（后续在节点执行的构建的工作目录）

  - Launch method 是启动方式，这里选择使用 SSH 方式
  - Host 填写节点的 IP 地址或者域名
  - Credentials 表示认证，点击Add创建一个并使用即可
    - Username：**节点的**用户名
    - Password：**节点的**登录密码
  - Host Key Verification Strategy：为了方便，这里选择 Non Verificating Verification Strategy 即可

- 连接

  点击 Save 后，Jenkins 会自动连接上节点，并在工作目录中注入 remoting.jar 等文件

  连接成功后，即可在 Jenkins 中看到节点



#### Jenkins 备份

- 直接备份Jenkins 目录
- 使用插件备份



#### 在项目中使用Jenkins  来构建 ：

- Jenkins 页面设置 pipeline  （我们把 pipeline 代码和 业务代码是放在不同仓库的 对服务来说没有侵入性）

- 项目中配置 Jenkinsfile 文件  （类似.gitlab-ci.yml）



发布流程：

起初我们为每个应用都写了一个 Jenkinsfile，里面的逻辑有拉取代码、编译应用、上传镜像到仓库和发布到 k8s 集群等。接着我为了结合现有的发布流程，通过 Jenkins 的动态参数实现了完全发布、制作镜像、发布配置、上线应用、回滚应用这样五种流程。

处理配置：

由于前面提到了 ConfigMap 不支持版本控制，因此配置中心拉取配置生成 ConfigMap 的事情就由 Jenkins 来实现了。我们会在 ConfigMap 名称后加上当前应用的版本号，将该版本的 ConfigMap 关联到 Deployment 中。这样在执行回滚应用时 ConfigMap 也可以一起回滚。同时 ConfigMap 的清理工作也可以在这里完成。



复用代码：

随着应用的增多，Jenkinsfile 也越来越多，如果要修改一个部署逻辑将会修改全部的 Jenkinsfile，这显然是不可接受的，于是我们开始优化 Jenkinsfile。



首先我们为不同类型的应用创建了不同的 yaml 模板，并用模板变量替换了里面的参数。接着我们使用了 Jenkins Shared Library 来编写通用的 CI/CD 逻辑，而 Jenkinsfile 里只需要填写需要执行什么逻辑和相应的参数即可。这样当一个逻辑需要变更时，我们直接修改通用库里的代码就全部生效了。





新建一个Pipeline项目，Web UI 写入Pipeline的构建脚本，对于单个项目来说，使用这样的Pipeline来构建能够满足绝大部分需求，但是这样做也有很多缺陷，包括：

- 多个项目的Pipeline打包脚本不能公用，导致一个项目写一份脚本，维护比较麻烦。一个变动，需要修改多个job的脚本；
- 多个人维护构建job的时候，可能会覆盖彼此的代码；
- 修改脚本失败以后，无法回滚到上个版本；
- 无法进行构建脚本的版本管理，老版本发修复版本需要构建，可能和现在用的job版本已经不一样了，等等。



既然存在缺陷，我们就要找更好的方式，其实Jenkins提供了一个更优雅的管理Pipeline脚本的方式，在配置项目Pipeline的时候，选择`Pipeline script from SCM`，如下图所示：



#### 用户管理

集成LDAP



#### Jenkins 插件下载加速

```
# 替换为清华大学的镜像源
cd /var/lib/jenkins/updates
sed -i 's/http:\/\/updates.jenkins-ci.org\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' default.json && sed -i 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' default.json
```

#### 配置Gitlab

集成gitlab，让jenkins能够直接读取修改gitlab中的代码，方便项目的构建

安装gitlab-plugin

在“系统管理” -> “系统设置“ -> “Gitlab” 中配置对应的gitlab信息

点击“Test Connection”测试下配置是否成功





#### 构建触发器

- 其他工程构建后触发
- 定时构建
- GitHub hook trigger for GITScm polling
- 轮询 SCM
- 关闭构建
- 静默期
- 触发远程构建 (例如,使用脚本)



我们使用 Poll SCM 作为构建触发器；设置这个选项来让 Jenkins 定期检查 Git 仓库（按 * * * * 指示的每分钟进行检查）。如果仓库从上次轮询后做了修改，任务就会被触发。

从流水线本身来说，我们指定了仓库的 URL 以及凭据。分支是 master 分支。

这个实验中，我们在一个 Jenkinsfile 中添加了所有的任务的代码，Jenkinsfile 跟代码一样存放在同一个仓库当中。Jenkinsfile 我们会在后面的文章中进行讨论。







---

- 1. 开发人员提交代码到 Gitlab 代码仓库
- 2. 通过 Gitlab 配置的 Jenkins Webhook 触发 Pipeline 自动构建
- 3. Jenkins 触发构建构建任务，根据 Pipeline 脚本定义分步骤构建
- 4. 先进行代码静态分析，单元测试
- 5. 然后进行 Maven 构建（Java 项目）
- 6. 根据构建结果构建 Docker 镜像
- 7. 推送 Docker 镜像到 Harbor 仓库
- 8. 触发更新服务阶段，使用 Helm 安装/更新 Release
- 9. 查看服务是否更新成功。




### Pipeline 中变量的使用 