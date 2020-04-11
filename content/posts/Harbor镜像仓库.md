---
title: Harbor镜像仓库的高可用
date: 2020-04-11
layout: post
---


Harbor 是基于 Docker Registry 的带UI的容器镜像仓库，界面友好，支持中文，现在应该是国内使用最多的开源容器镜像仓库了。



Harbor 高可用的方式

- 多实例共享后端存储(采取挂载文件系统方式：可以是S3，Ceph，OSS）
- 多实例相互数据同步(基于镜像复制模式)



**官网推荐的Harbor 高可用安装方式 是采用Helm 来安装**

```
# 添加helm 仓库，把chart 下载到当前路径
helm repo add harbor https://helm.goharbor.io
helm fetch harbor/harbor --untar

#配置


#安装
# Install the Harbor helm chart with a release name my-release:
#Helm 3:
helm install my-release .
```





Harbor 后端存储采用阿里云的OSS，可以解决容器镜像空间越来越大的问题，也提高了存储的可用性

在harbor.yml 中添加

```
#注意REGISTRY_STORAGE_OSS_ENDPOINT的值一定要写成OSS的访问域名，不能写成OSS的EndPoint
storage_service:
  oss:
    accesskeyid: xxxxx
    accesskeysecret: XXXX
    region: oss-cn-shanghai
    endpoint: xxx.oss-cn-shanghai.aliyuncs.com
    internal: true
    bucket: xxx
    encrypt: false
    secure: false
    chunksize: 5242880
    rootdirectory:
```











### 开启clair 扫描功能

Harbor 镜像仓库使用[Clair](https://github.com/coreos/clair) 来做镜像静态扫描

>   	在 Harbor 中，我们集成了开源项目 Clair 的扫描功能，可从公开的 CVE 字典库下载漏洞资料。CVE 是 Common Vulnerabilities and Exposures 的缩写，由一些机构自愿参与维护的软件安全漏洞标识，记录已知的漏洞标准描述及相关信息，公众可以免费获取和使用这些信息。全球共有77个机构参与维护不同软件的 CVE 库，例如：VMware 维护着 VMware 产品的 CVE 库，红帽维护着Linux 上的 CVE 等等。容器镜像基本上涉及的是 Linux 操作系统上的软件，因此镜像扫描需要参考 Linux 相关的 CVE 库，目前 Harbor（Clair）使用的CVE。

安装的时候，可选安装clair 扫描插件功能

```sh
./install.sh --with-clair
```





#### Harbor 更改配置重启

```
# 停止 Harbor
docker-compose down -v
# 更改配置
vim harbor.yml
# 可选 Notary，Clair，chart repository service
./prepare --with-notary --with-clair --with-chartmuseum
#重新创建并启动Harbor实例
docker-compose up -d
```

