---
title: Kubernetes集群安装
date: 2019-07-20
layout: post
tags: 
---


### Master包含组件

Master 组件提供的集群控制，Master 组件对集群做出全局性决策(例如：调度)等，负责维护集群的目标状态。

| 组件                    | 说明                                                         |
| ----------------------- | ------------------------------------------------------------ |
| etcd                    | 用于 Kubernetes 的后端数据存储，所有集群数据都存储在此处     |
| kube-apiserver          | 对外暴露了 Kubernetes API，它是的 Kubernetes 前端控制层，只有 API Server 会与 etcd 通信，其它模块都必须通过 API Server 访问集群状态。 |
| kube-controller-manager | 处理集群中常规任务，它是单独的进程，内部包含多个控制器，包含节点控制器,副本控制器,端点控制器,服务帐户和令牌控制器 |
| kube-scheduler          | 监视新创建的 Pod 为新创建的 POD 分配合适的 node 节点         |
| addons（插件）          | 插件是实现集群功能的 Pod 和 Service，一般被创建于 kube-system 命名空间。例如：coreDNS |




### Node  包含组件 

Node 节点实际负责实施，也就是运行 POD 的节点，上面运行的组件有

| 组件       | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| kubelet    | 它监测已经分配给自己的 Pod，为 POD 准备卷，下载 POD 所需的 Secret，下载镜像并运行，进行生命周期探测，上报 POD 和节点状态 |
| kube-proxy | 通过维护主机上的网络规则并执行连接转发，将 Kubernetes 提供的网络服务代理到每个节点上，实现了Kubernetes服务抽象 |
| docker     | 用于运行容器                                                 |




```
#!/bin/bash
#执行该脚本需要添加k8s版本号,作为参数 将脚本第79行（已高亮）的 ${1} 替换成您需要的版本号，例如 1.17.0
# 在 master 节点和 worker 节点都要执行

# 安装 docker
# 参考文档如下
# https://docs.docker.com/install/linux/docker-ce/centos/ 
# https://docs.docker.com/install/linux/linux-postinstall/

# 卸载旧版本
yum remove -y docker \
docker-client \
docker-client-latest \
docker-common \
docker-latest \
docker-latest-logrotate \
docker-logrotate \
docker-selinux \
docker-engine-selinux \
docker-engine

# 设置 yum repository
yum install -y yum-utils \
device-mapper-persistent-data \
lvm2
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 安装并启动 docker
yum install -y docker-ce-18.09.7 docker-ce-cli-18.09.7 containerd.io
systemctl enable docker
systemctl start docker

# 安装 nfs-utils
# 必须先安装 nfs-utils 才能挂载 nfs 网络存储
yum install -y nfs-utils
yum install -y wget

# 关闭 防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关闭 SeLinux
setenforce 0
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config

# 关闭 swap
swapoff -a
yes | cp /etc/fstab /etc/fstab_bak
cat /etc/fstab_bak |grep -v swap > /etc/fstab

# 修改 /etc/sysctl.conf
# 如果有配置，则修改
sed -i "s#^net.ipv4.ip_forward.*#net.ipv4.ip_forward=1#g"  /etc/sysctl.conf
sed -i "s#^net.bridge.bridge-nf-call-ip6tables.*#net.bridge.bridge-nf-call-ip6tables=1#g"  /etc/sysctl.conf
sed -i "s#^net.bridge.bridge-nf-call-iptables.*#net.bridge.bridge-nf-call-iptables=1#g"  /etc/sysctl.conf
# 可能没有，追加
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-ip6tables = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 1" >> /etc/sysctl.conf
# 执行命令以应用
sysctl -p

# 配置K8S的yum源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 卸载旧版本
yum remove -y kubelet kubeadm kubectl

# 安装kubelet、kubeadm、kubectl
yum install -y kubelet-${1} kubeadm-${1} kubectl-${1}

# 修改docker Cgroup Driver为systemd
# # 将/usr/lib/systemd/system/docker.service文件中的这一行 ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
# # 修改为 ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --exec-opt native.cgroupdriver=systemd
# 如果不修改，在添加 worker 节点时可能会碰到如下错误
# [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". 
# Please follow the guide at https://kubernetes.io/docs/setup/cri/
sed -i "s#^ExecStart=/usr/bin/dockerd.*#ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --exec-opt native.cgroupdriver=systemd#g" /usr/lib/systemd/system/docker.service

# 设置 docker 镜像，提高 docker 镜像下载速度和稳定性
# 如果您访问 https://hub.docker.io 速度非常稳定，亦可以跳过这个步骤
curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://f1361db2.m.daocloud.io

# 重启 docker，并启动 kubelet
systemctl daemon-reload
systemctl restart docker
systemctl enable kubelet && systemctl start kubelet

docker version
```



<a name="D3lWQ"></a>

### 初始化 Master 节点

```
# 只在 master 节点执行
# 替换 x.x.x.x 为 master 节点的内网IP
# export 命令只在当前 shell 会话中有效，开启新的 shell 窗口后，如果要继续安装过程，请重新执行此处的 export 命令
export MASTER_IP=x.x.x.x
# 替换 apiserver.demo 为 您想要的 dnsName
export APISERVER_NAME=apiserver.demo
# Kubernetes 容器组所在的网段，该网段安装完成后，由 kubernetes 创建，事先并不存在于您的物理网络中
export POD_SUBNET=10.100.0.1/16
echo "${MASTER_IP}    ${APISERVER_NAME}" >> /etc/hosts
```

### 请将脚本第21行（已高亮）的 ${1} 替换成您需要的版本号，例如 1.17.0

```
#!/bin/bash

# 只在 master 节点执行

# 脚本出错时终止执行
set -e

if [ ${#POD_SUBNET} -eq 0 ] || [ ${#APISERVER_NAME} -eq 0 ]; then
  echo -e "\033[31;1m请确保您已经设置了环境变量 POD_SUBNET 和 APISERVER_NAME \033[0m"
  echo 当前POD_SUBNET=$POD_SUBNET
  echo 当前APISERVER_NAME=$APISERVER_NAME
  exit 1
fi


# 查看完整配置选项 https://godoc.org/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/v1beta2
rm -f ./kubeadm-config.yaml
cat <<EOF > ./kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v${1}
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
controlPlaneEndpoint: "${APISERVER_NAME}:6443"
networking:
  serviceSubnet: "10.96.0.0/16"
  podSubnet: "${POD_SUBNET}"
  dnsDomain: "cluster.local"
EOF

# kubeadm init
# 根据您服务器网速的情况，您需要等候 3 - 10 分钟
kubeadm init --config=kubeadm-config.yaml --upload-certs

# 配置 kubectl
rm -rf /root/.kube/
mkdir /root/.kube/
cp -i /etc/kubernetes/admin.conf /root/.kube/config

# 安装 calico 网络插件
# 参考文档 https://docs.projectcalico.org/v3.10/getting-started/kubernetes/
echo "安装calico-3.10.2"
rm -f calico-3.10.2.yaml
wget https://kuboard.cn/install-script/calico/calico-3.10.2.yaml
sed -i "s#192\.168\.0\.0/16#${POD_SUBNET}#" calico-3.10.2.yaml
kubectl apply -f calico-3.10.2.yaml
```

<a name="AnGTK"></a>

#### 检查Master 初始化结果

```
# 只在 master 节点执行

# 执行如下命令，等待 3-10 分钟，直到所有的容器组处于 Running 状态
watch kubectl get pod -n kube-system -o wide

# 查看 master 节点初始化结果
kubectl get nodes -o wide
```


<a name="9imZK"></a>

### 初始化  work 节点

```
# 只在 master 节点执行
kubeadm token create --print-join-command
```


```
# 只在 worker 节点执行
# 替换 x.x.x.x 为 master 节点的内网 IP
export MASTER_IP=x.x.x.x
# 替换 apiserver.demo 为初始化 master 节点时所使用的 APISERVER_NAME
export APISERVER_NAME=apiserver.demo
echo "${MASTER_IP}    ${APISERVER_NAME}" >> /etc/hosts

# 替换为 master 节点上 kubeadm token create 命令的输出
kubeadm join apiserver.demo:6443 --token mpfjma.4vjjg8flqihor4vt     --discovery-token-ca-cert-hash sha256:6f7a8e40a810323672de5eee6f4d19aa2dbdb38411845a1bf5dd63485c43d303
```


<a name="lmnJc"></a>

#### 检查初始化结果

```
# 只在 master 节点执行
kubectl get nodes -o wide
```

输出结果如下所示

```
[root@demo-master-a-1 ~]# kubectl get nodes
NAME     STATUS   ROLES    AGE     VERSION
demo-master-a-1   Ready    master   5m3s    v1.17.x
demo-worker-a-1   Ready    <none>   2m26s   v1.17.x
demo-worker-a-2   Ready    <none>   3m56s   v1.17.x
```