---
title: Harbor镜像仓库的高可用
date: 2020-04-11
layout: post
---


Harbor 是基于 Docker Registry 的带UI的容器镜像仓库，界面友好，支持中文，现在应该是国内使用最多的开源容器镜像仓库了。

#### Harbor 安装

参考官网 Docker-composer 或者Helm 安装

#### Harbor 高可用的方式

- 多个Harbor实例，前端加一层负载均衡，后端使用相同的存储（OSS，NFS）与数据库（pg，redis）    该方式是把Harbor 无状态的部分抽离出来
- 多实例共享后端存储(采取挂载文件系统方式：可以是S3，Ceph，OSS）
- 多实例相互数据同步(基于镜像复制模式)



![image-20200421173543123](https://blog-pic-1253367462.cos.ap-shanghai.myqcloud.com/image-20200421173543123.png)



使用外部数据库&Redis

```
#禁用本地数据库

#使用外部数据库 （只支持PostgreSQL）
#您必须为Harbor core，Clair，Notary服务器和Notary签署者创建四个数据库。该表在Harbor启动时自动生成。
external_database

#配置外部Redis实例
external_redis
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

配置参考：https://docs.docker.com/registry/storage-drivers/oss/



#### 使用Https

- 在Harbor 前端Nginx层 放SSL 证书

  我们在Harbor前面再放置一个nginx作为接入层，在这个nginx上配置SSL，可以使用自签名证书，如果绑定了域名和固定IP，可以申请[使用 Let's Encrypt的免费HTTPS证书](https://blog.frognew.com/2017/05/lets-encrypt-https-nginx.html)。

  下面的是前置nginx上做负载均衡和SSL termination的nginx配置文件片段：

  ```
  upstream harbor {
      server 192.168.61.11:8090 weight=1 max_fails=2 fail_timeout=30s;
      server 192.168.61.12:8090 weight=1 max_fails=2 fail_timeout=30s;
  }
  
  server {
      listen       443 ssl;
      server_name  harbor.frognew.com;
      ssl_protocols  TLSv1 TLSv1.1 TLSv1.2;
      ssl_certificate /etc/letsencrypt/live/harbor.frognew.com/fullchain.pem;
      ssl_certificate_key /etc/letsencrypt/live/harbor.frognew.com/privkey.pem;
      location / {
          proxy_pass   http://harbor;
          proxy_set_header   Host             $host;
          proxy_set_header   X-Real-IP        $remote_addr;
          proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
          proxy_set_header   X-Forwarded-Proto https;
          client_max_body_size 300M;
      }
  }
  
  server {
      listen 80;
      server_name harbor.frognew.com;
      rewrite ^(.*)$  https://$host$1 permanent;
  }
  
  ```

  

- 配置 SSL 证书

  

> **重要信息**：Harbor不附带任何证书。在1.9.x及以下版本中，默认情况下，Harbor使用HTTP服务于注册表请求。这仅在空白的测试或开发环境中可接受。在生产环境中，请始终使用HTTPS。如果您启用Content Trust with Notary来正确签名所有图像，则必须使用HTTPS。
>
> 您可以使用由受信任的第三方CA签名的证书，也可以使用自签名证书。有关如何创建CA以及如何使用CA签署服务器证书和客户端证书的信息，请参阅“ [使用HTTPS访问配置Harbor”](https://goharbor.io/docs/1.10/install-config/configure-https/)。



### 开启clair 扫描功能

Harbor 镜像仓库使用[Clair](https://github.com/coreos/clair) 来做镜像静态扫描

>   		在 Harbor 中，我们集成了开源项目 Clair 的扫描功能，可从公开的 CVE 字典库下载漏洞资料。CVE 是 Common Vulnerabilities and Exposures 的缩写，由一些机构自愿参与维护的软件安全漏洞标识，记录已知的漏洞标准描述及相关信息，公众可以免费获取和使用这些信息。全球共有77个机构参与维护不同软件的 CVE 库，例如：VMware 维护着 VMware 产品的 CVE 库，红帽维护着Linux 上的 CVE 等等。容器镜像基本上涉及的是 Linux 操作系统上的软件，因此镜像扫描需要参考 Linux 相关的 CVE 库，目前 Harbor（Clair）使用的CVE。

安装的时候，可选安装clair 扫描插件功能

```sh
./install.sh --with-clair
```

>  clair扫描没有漏洞，阿里云镜像仓库扫描有漏洞的情况



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



#### Harbor 垃圾清理

镜像签名

![image-20200408192824036](https://blog-pic-1253367462.cos.ap-shanghai.myqcloud.com/image-20200408192824036.png)



