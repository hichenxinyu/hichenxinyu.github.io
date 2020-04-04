---
title: Prometheus+Grafana监控实践
date: 2019-07-01
layout: post
---

#### 什么是Prometheus
>Prometheus 是由 SoundCloud 开源监控告警解决方案，从 2012 年开始编写代码，再到 2015 年 github 上开源以来，已经吸引了 9k+ 关注，以及很多大公司的使用；2016 年 Prometheus 成为继 k8s 后，第二名 CNCF(Cloud Native Computing Foundation) 成员。

作为新一代开源解决方案，很多理念与 Google SRE 运维之道不谋而合。


#### 为什么选择prometheus：
- Prometheus 是按照 Google SRE 运维之道的理念构建的，具有实用性和前瞻性。
- Prometheus 社区非常活跃，基本稳定在 1个月1个版本的迭代速度，从 2016 年 v1.01 开始接触使用以来，到目前发布的 v1.8.2 以及最新最新的 v2.1 ，你会发现 Prometheus 一直在进步、在优化。
- Go 语言开发，性能不错，安装部署简单，多平台部署兼容性好。
- 丰富的数据收集客户端，官方提供了各种常用 exporter。
- 丰富强大的查询能力,内置更强大的统计函数。
- Prometheus 属于一站式监控告警平台，依赖少，功能齐全。
- Prometheus支持对云或容器的监控，其他监控系统主要对主机监控。


<!-- more -->
Prometheus 在数据存储扩展性以及持久性上没有 InfluxDB，OpenTSDB，Sensu 好。


#### 安装prometheus

1. 下载二进制包
```
在这个页面https://prometheus.io/download
找到最新的二进制包
wget https://github.com/prometheus/prometheus/releases/download/v2.7.2/prometheus-2.7.2.linux-amd64.tar.gz
```


2. 解压并进入对应文件夹
```
tar xvfz prometheus-*.tar.gz
cd prometheus-*
```
3. 启动prometheus
```
./prometheus --config.file=prometheus.yml
```



grafana dashboard
主机基础监控 dashboard id 9276






##### 此时，prometheus 服务已在后台运行，注意暴露的端口号 9090（address=0.0.0.0:9090），可以直接在浏览器打开 ip:9090 查看简易控制台，此控制台可以做调试使用，需要更强大、美观的数据展示官方建议使用 Grafana.

----

#### 安装 grafana 
（数据展示<https://grafana.com>）

```
wget https://s3-us-west-2.amazonaws.com/grafana-releases/release/grafana-5.0.2-1.x86_64.rpm

yum  localinstall grafana-5.0.2-1.x86_64.rpm

service grafana-server start
```

此时，grafana 服务已经启动，可以直接在浏览器中打开 localhost:3000 开始体验 grafana 了








----


####  exporter 

Prometheus官网有很多exporter，exporter可以理解为**采集客户端**，即采集指标，通过http方式暴露metrics给prometheus，Prometheus通过pull方式拉取这些指标。


1. node_exporter（linux 系统信息采集器）

```
wget https://github.com/prometheus/node_exporter/releases/download/v0.18.0/node_exporter-0.18.0.linux-amd64.tar.gz

tar -zxvf node_exporter-0.18.0.linux-amd64.tar.gz

cd node_exporter-0.18.0.linux-amd64

nohup ./node_exporter & （在后台运行系统信息采集器）
 ```
 可以通过curl查看  
 curl 127.0.0.1:9100/metrics

> Tips：0.16.0 版本在 centos 6.x 运行时报错 kernel too old，下载 0.15.2 或之前版本即可
https://github.com/prometheus/node_exporter/releases/download/v0.15.2/node_exporter-0.15.2.linux-amd64.tar.gz

此时，系统信息采集器已在后台运行，等待 prometheus 服务来拉取数据，注意暴露的端口号 9100（msg="Listening on :9100"），prometheus 拉取数据需要使用此端口号




#### 安装 prometheus 
prometheus <https://prometheus.io/>）
```
wget https://github.com/prometheus/prometheus/releases/download/v2.2.1/prometheus-2.2.1.linux-amd64.tar.gz
tar -zxvf prometheus-2.2.1.linux-amd64.tar.gz
cd prometheus-2.2.1.linux-amd64
```
vi prometheus.yml

```
global:
  scrape_interval: 10s
  evaluation_interval: 10s

scrape_configs:
  - job_name: prometheus
    scrape_interval: 10s
    static_configs:
      - targets: ['localhost:9090']
        labels:
          instance: prometheus

  - job_name: mysql
    scrape_interval: 10s
    static_configs:
      - targets: ['localhost:9104']
        labels:
          instance: mysql

  - job_name: 'linux'
    scrape_interval: 10s
    static_configs:
      - targets: ['localhost:9100']
        labels:
          instance: linux
```

##### 启动
nohup ./prometheus --config.file="prometheus.yml" &（在后台运行 prometheus 服务）




#### Prometheus配置的热加载
Prometheus配置信息的热加载有两种方式：
- 第一种热加载方式：查看Prometheus的进程id，发送SIGHUP信号:
kill -HUP <pid>

- 第二种热加载方式：发送一个POST请求到/-/reload，需要在启动时给定--web.enable-lifecycle选项：
curl -X POST http://localhost:9090/-/reload


我们使用的是第一种热加载方式，systemd unit文件如下：
```
[Unit]
Description=prometheus
After=network.target
[Service]
Type=simple
User=prometheus
ExecStart=/usr/local/prometheus/prometheus \
 --config.file=/usr/local/prometheus/prometheus.yml \
 --storage.tsdb.path=/home/prometheus/data \
 --storage.tsdb.retention=365d \
 --web.listen-address=:9090 \
 --web.external-url=https://prometheus.frognew.com
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
[Install]
WantedBy=multi-user.target

```

systemctl reload prometheus

#### 告警服务 Alertmanager
告警能力在Prometheus的架构中被划分成两个独立的部分。如下所示，通过在Prometheus中定义AlertRule（告警规则），Prometheus会周期性的对告警规则进行计算，如果满足告警触发条件就会向Alertmanager发送告警信息。
![2957ddc2799240dd958cbdf9bf4a46f7.png](evernotecid://94C76643-58DA-4407-AFA8-9F2A33EF0AAA/appyinxiangcom/11651347/ENResource/p3081)


#### 一些问题 

1. 为什么选择 pull  而不是push
通过HTTP提供了许多优势：
- 您可以在开发更改时在笔记本电脑上运行监控。
- 您可以更轻松地判断目标是否已关闭。
- 您可以手动转到目标并使用Web浏览器检查其运行状况。
总体而言，我们认为拉动比推动略好，但在考虑监控系统时，不应将其视为重点。




附录:
线上 Prometheus 配置文件 prometheus.yml
```
# my global config
global:
  scrape_interval:     30s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 30s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
       - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
   - "/etc/prometheus/rules/*.rules"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']

  - job_name: 'telegraf'
    file_sd_configs:
      - files: ['./*.json']

remote_write:
  - url: "http://influxdb:8086/api/v1/prom/write?db=prometheus"

remote_read:
  - url: "http://influxdb:8086/api/v1/prom/read?db=prometheus"
```
