---
title: Gitlab CI/CD
date: 2019-06-12
layout: post
---

GitLab CI/CD 是gitlab8.0之后自带的一个持续集成系统，中心思想是当每一次push到gitlab的时候，都会触发一次脚本执行，然后脚本的内容包括了测试，编译，部署等一系列自定义的内容。

GitLab CI/CD的脚本执行，需要自定义安装对应gitlab-runner来执行，代码push之后，webhook检测到代码变化，就会触发gitlab-CI，分配到各个Runner来运行相应的脚本script。这些脚本有的是测试项目用的，有的是部署用的。

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

