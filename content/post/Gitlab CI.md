# GitLab CI/CD

GitLab CI / CD是GitLab内置的工具，用于通过[连续方法](https://docs.gitlab.com/ce/ci/introduction/index.html#introduction-to-cicd-methodologies)进行软件开发：

- 持续集成（CI）
- 连续交付（CD）
- 持续部署（CD）



GitLab CI/CD 是gitlab8.0之后自带的一个持续集成系统，中心思想是当每一次push到gitlab的时候，都会触发一次脚本执行，然后脚本的内容包括了测试，编译，部署等一系列自定义的内容。

GitLab CI/CD的脚本执行，需要自定义安装对应gitlab-runner来执行，gitlab-runner 默认每三秒去调用gitlab api，发现代码有更新就会触发gitlab-CI，分配到各个Runner来运行相应的脚本script。这些脚本有的是测试项目用的，有的是部署用的。



### GitLab CI/CD 优势

- 轻量级，不需要复杂的安装手段。
- 配置简单，与gitlab可直接适配。
- 实时构建日志十分清晰，UI交互体验很好
- 使用 YAML 进行配置，任何人都可以很方便的使用。

### GitLab CI/CD 劣势

- 没有统一的管理界面，无法统筹管理所有项目
- 配置依赖于代码仓库，耦合度没有Jenkins低



gitlab-runner 安装(个人比较喜欢 yum or rpm 安装软件)

```
#添加官方 yum 源
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh | sudo bash
#安装 gitlab-runner 
yum install gitlab-runner -y
```

gitlab-runner 注册 

```
gitlab-ci-multi-runner register
#然后依次填入  gitlab url，Token，Runner 类型
```



#### 变量

获得提交时间戳

```
before_script:
  - export COMMIT_TIME=$(git show -s --format=%ct $CI_COMMIT_SHA)  
```





#### 镜像Tag

CI_PIPELINE_ID

![image-20200409164706457](https://blog-pic-1253367462.cos.ap-shanghai.myqcloud.com/image-20200409164706457.png)



可以使用  分支+日期+Pipeline序列号，比如 master-200401-12743     master分支在20年4月1日的序列号为12743的Pipeline 上构建的 







### 一些坑

Docker 权限问题

构建镜像时报错 

gitlab ci  docker build  permission denied

解决方案:

在gitlab-runner机器上授权

```
usermod -aG docker gitlab-runner
sudo service docker restart
```







