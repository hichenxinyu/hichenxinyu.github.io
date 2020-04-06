---
title: ElasticSearch 压测
date: 2019-05-22
categories:
  - ELK
layout: post
---

#### 本次压测目标：

- 测试集群读写性能 & 为集群容量规划提供数据支撑
- 测试 ES 新版本，结合实际场景与老版本进行比较，评估是否进行升级
- 性能优化&性能问题诊断
- 对 ES 配置参数进行修改，评估优化效果（长期）
- 确定系统稳定性，考察系统功能极限与稳定性
  #### 

#### 可选择的压测工具

- Elastic 官方压测工具：rally   
- 基于HTTP 的压测第三方压测工具：JMeter 等

#### 压测流程如下

- 压测计划
- 脚本开发
- 压测环境搭建&压测开始
- 结果分析

---


- 为方便对比，压测数据统一使用 geonames 数据集（大小3.2G，共11523468个文档）
- 时间： 完整的一次压测过程，时间以小时计

#### 安装

依赖

- 3.5+ including pip3 
- git 1.9+
- JDK 

```
pip3 install esrally
```

#### Elastic 官方压测结果[链接](https://elasticsearch-benchmarks.elastic.co/)

```
https://elasticsearch-benchmarks.elastic.co/
```

#### 配置

```
 esrally configure --advanced-config
```


只测试 1000 条数据

```
esrally --distribution-version=5.6.16 --test-mode
```

开始测试各版本单机性能

```
esrally race --distribution-version=5.6.16 --track=geonames --challenge=append-no-conflicts --user-tag="version:5.6.16"
```

#### 

#### 查看本地 tracks

```
esrally list tracks
```

#### 

#### 压测现有集群

```
esrally race --pipeline=benchmark-only --target-hosts=10.0.19.218:9200 --distribution-version=6.7.2  --track=geonames --challenge=append-no-conflicts
```


#### 测试环境（10.0.19.218）先跑一遍

```
esrally race --pipeline=benchmark-only --target-hosts=10.0.19.218:9200 --distribution-version=6.7.2  --track=geonames --challenge=append-no-conflicts
```

##### 结果

```
|                                                         Metric |                   Task |    Value |    Unit |
|---------------------------------------------------------------:|-----------------------:|---------:|--------:|
|                     Cumulative indexing time of primary shards |                        |  157.082 |     min |
|             Min cumulative indexing time across primary shards |                        |        0 |     min |
|          Median cumulative indexing time across primary shards |                        | 0.256483 |     min |
|             Max cumulative indexing time across primary shards |                        |  30.3196 |     min |
|            Cumulative indexing throttle time of primary shards |                        |        0 |     min |
|    Min cumulative indexing throttle time across primary shards |                        |        0 |     min |
| Median cumulative indexing throttle time across primary shards |                        |        0 |     min |
|    Max cumulative indexing throttle time across primary shards |                        |        0 |     min |
|                        Cumulative merge time of primary shards |                        |  161.364 |     min |
|                       Cumulative merge count of primary shards |                        |    23823 |         |
|                Min cumulative merge time across primary shards |                        |        0 |     min |
|             Median cumulative merge time across primary shards |                        |  1.43677 |     min |
|                Max cumulative merge time across primary shards |                        |  23.3528 |     min |
|               Cumulative merge throttle time of primary shards |                        | 0.933833 |     min |
|       Min cumulative merge throttle time across primary shards |                        |        0 |     min |
|    Median cumulative merge throttle time across primary shards |                        |        0 |     min |
|       Max cumulative merge throttle time across primary shards |                        |   0.1983 |     min |
|                      Cumulative refresh time of primary shards |                        |  59.8701 |     min |
|                     Cumulative refresh count of primary shards |                        |   254455 |         |
|              Min cumulative refresh time across primary shards |                        |        0 |     min |
|           Median cumulative refresh time across primary shards |                        |  1.47577 |     min |
|              Max cumulative refresh time across primary shards |                        |  7.86888 |     min |
|                        Cumulative flush time of primary shards |                        |  1.82032 |     min |
|                       Cumulative flush count of primary shards |                        |       76 |         |
|                Min cumulative flush time across primary shards |                        |        0 |     min |
|             Median cumulative flush time across primary shards |                        |  0.00315 |     min |
|                Max cumulative flush time across primary shards |                        | 0.473867 |     min |
|                                             Total Young Gen GC |                        |  156.258 |       s |
|                                               Total Old Gen GC |                        |    0.504 |       s |
|                                                     Store size |                        |  10.2503 |      GB |
|                                                  Translog size |                        |  3.40687 |      GB |
|                                         Heap used for segments |                        |  28.6233 |      MB |
|                                       Heap used for doc values |                        |  1.08705 |      MB |
|                                            Heap used for terms |                        |  25.1319 |      MB |
|                                            Heap used for norms |                        | 0.283142 |      MB |
|                                           Heap used for points |                        | 0.717692 |      MB |
|                                    Heap used for stored fields |                        |  1.40359 |      MB |
|                                                  Segment count |                        |      394 |         |
|                                                 Min Throughput |           index-append |  13680.2 |  docs/s |
|                                              Median Throughput |           index-append |  15924.4 |  docs/s |
|                                                 Max Throughput |           index-append |  17251.8 |  docs/s |
|                                        50th percentile latency |           index-append |  1917.34 |      ms |
|                                        90th percentile latency |           index-append |  3282.31 |      ms |
|                                        99th percentile latency |           index-append |  4744.88 |      ms |
|                                      99.9th percentile latency |           index-append |  6111.59 |      ms |
|                                       100th percentile latency |           index-append |   6868.8 |      ms |
|                                   50th percentile service time |           index-append |  1917.34 |      ms |
|                                   90th percentile service time |           index-append |  3282.31 |      ms |
|                                   99th percentile service time |           index-append |  4744.88 |      ms |
|                                 99.9th percentile service time |           index-append |  6111.59 |      ms |
|                                  100th percentile service time |           index-append |   6868.8 |      ms |
|                                                     error rate |           index-append |        0 |       % |
|                                                 Min Throughput |            index-stats |    47.27 |   ops/s |
|                                              Median Throughput |            index-stats |    47.77 |   ops/s |
|                                                 Max Throughput |            index-stats |    47.88 |   ops/s |
|                                        50th percentile latency |            index-stats |  9808.46 |      ms |
|                                        90th percentile latency |            index-stats |  13753.5 |      ms |
|                                        99th percentile latency |            index-stats |  14571.1 |      ms |
|                                      99.9th percentile latency |            index-stats |  14645.1 |      ms |
|                                       100th percentile latency |            index-stats |  14653.5 |      ms |
|                                   50th percentile service time |            index-stats |   20.069 |      ms |
|                                   90th percentile service time |            index-stats |  23.2235 |      ms |
|                                   99th percentile service time |            index-stats |  28.6327 |      ms |
|                                 99.9th percentile service time |            index-stats |  46.0017 |      ms |
|                                  100th percentile service time |            index-stats |  48.4586 |      ms |
|                                                     error rate |            index-stats |        0 |       % |
|                                                 Min Throughput |             node-stats |    51.61 |   ops/s |
|                                              Median Throughput |             node-stats |    54.58 |   ops/s |
|                                                 Max Throughput |             node-stats |    54.95 |   ops/s |
|                                        50th percentile latency |             node-stats |  4375.34 |      ms |
|                                        90th percentile latency |             node-stats |  7131.76 |      ms |
|                                        99th percentile latency |             node-stats |  7723.09 |      ms |
|                                      99.9th percentile latency |             node-stats |  7780.35 |      ms |
|                                       100th percentile latency |             node-stats |  7785.54 |      ms |
|                                   50th percentile service time |             node-stats |  17.4001 |      ms |
|                                   90th percentile service time |             node-stats |  21.1093 |      ms |
|                                   99th percentile service time |             node-stats |  23.9763 |      ms |
|                                 99.9th percentile service time |             node-stats |  34.4598 |      ms |
|                                  100th percentile service time |             node-stats |  37.7474 |      ms |
|                                                     error rate |             node-stats |        0 |       % |
|                                                 Min Throughput |                default |    49.87 |   ops/s |
|                                              Median Throughput |                default |    49.99 |   ops/s |
|                                                 Max Throughput |                default |    50.02 |   ops/s |
|                                        50th percentile latency |                default |  22.9061 |      ms |
|                                        90th percentile latency |                default |  35.2368 |      ms |
|                                        99th percentile latency |                default |  56.4199 |      ms |
|                                      99.9th percentile latency |                default |  60.2163 |      ms |
|                                       100th percentile latency |                default |  62.0121 |      ms |
|                                   50th percentile service time |                default |  21.0608 |      ms |
|                                   90th percentile service time |                default |  23.0526 |      ms |
|                                   99th percentile service time |                default |  24.0483 |      ms |
|                                 99.9th percentile service time |                default |   26.301 |      ms |
|                                  100th percentile service time |                default |  27.9663 |      ms |
|                                                     error rate |                default |        0 |       % |
|                                                 Min Throughput |                   term |   200.07 |   ops/s |
|                                              Median Throughput |                   term |   200.08 |   ops/s |
|                                                 Max Throughput |                   term |   200.14 |   ops/s |
|                                        50th percentile latency |                   term |  2.71801 |      ms |
|                                        90th percentile latency |                   term |  3.03285 |      ms |
|                                        99th percentile latency |                   term |  11.4885 |      ms |
|                                      99.9th percentile latency |                   term |  26.8219 |      ms |
|                                       100th percentile latency |                   term |  28.1828 |      ms |
|                                   50th percentile service time |                   term |  2.56728 |      ms |
|                                   90th percentile service time |                   term |  2.85599 |      ms |
|                                   99th percentile service time |                   term |  3.55974 |      ms |
|                                 99.9th percentile service time |                   term |  8.51489 |      ms |
|                                  100th percentile service time |                   term |  26.6701 |      ms |
|                                                     error rate |                   term |        0 |       % |
|                                                 Min Throughput |                 phrase |   199.27 |   ops/s |
|                                              Median Throughput |                 phrase |   200.03 |   ops/s |
|                                                 Max Throughput |                 phrase |   200.06 |   ops/s |
|                                        50th percentile latency |                 phrase |  4.25964 |      ms |
|                                        90th percentile latency |                 phrase |  13.8712 |      ms |
|                                        99th percentile latency |                 phrase |  35.8641 |      ms |
|                                      99.9th percentile latency |                 phrase |  39.0226 |      ms |
|                                       100th percentile latency |                 phrase |  39.8982 |      ms |
|                                   50th percentile service time |                 phrase |   4.0484 |      ms |
|                                   90th percentile service time |                 phrase |  4.60117 |      ms |
|                                   99th percentile service time |                 phrase |  8.69007 |      ms |
|                                 99.9th percentile service time |                 phrase |   33.045 |      ms |
|                                  100th percentile service time |                 phrase |  38.8927 |      ms |
|                                                     error rate |                 phrase |        0 |       % |
|                                                 Min Throughput |   country_agg_uncached |     2.35 |   ops/s |
|                                              Median Throughput |   country_agg_uncached |     2.36 |   ops/s |
|                                                 Max Throughput |   country_agg_uncached |     2.37 |   ops/s |
|                                        50th percentile latency |   country_agg_uncached |  43731.8 |      ms |
|                                        90th percentile latency |   country_agg_uncached |    50463 |      ms |
|                                        99th percentile latency |   country_agg_uncached |  52056.4 |      ms |
|                                       100th percentile latency |   country_agg_uncached |  52232.7 |      ms |
|                                   50th percentile service time |   country_agg_uncached |  407.051 |      ms |
|                                   90th percentile service time |   country_agg_uncached |  520.447 |      ms |
|                                   99th percentile service time |   country_agg_uncached |  588.956 |      ms |
|                                  100th percentile service time |   country_agg_uncached |  650.993 |      ms |
|                                                     error rate |   country_agg_uncached |        0 |       % |
|                                                 Min Throughput |     country_agg_cached |    99.69 |   ops/s |
|                                              Median Throughput |     country_agg_cached |   100.06 |   ops/s |
|                                                 Max Throughput |     country_agg_cached |   100.13 |   ops/s |
|                                        50th percentile latency |     country_agg_cached |  3.25071 |      ms |
|                                        90th percentile latency |     country_agg_cached |  3.58457 |      ms |
|                                        99th percentile latency |     country_agg_cached |  6.40293 |      ms |
|                                      99.9th percentile latency |     country_agg_cached |  32.8548 |      ms |
|                                       100th percentile latency |     country_agg_cached |  38.2042 |      ms |
|                                   50th percentile service time |     country_agg_cached |  3.09234 |      ms |
|                                   90th percentile service time |     country_agg_cached |  3.40055 |      ms |
|                                   99th percentile service time |     country_agg_cached |  4.56434 |      ms |
|                                 99.9th percentile service time |     country_agg_cached |  13.7336 |      ms |
|                                  100th percentile service time |     country_agg_cached |  38.0936 |      ms |
|                                                     error rate |     country_agg_cached |        0 |       % |
|                                                 Min Throughput |                 scroll |    20.02 | pages/s |
|                                              Median Throughput |                 scroll |    20.02 | pages/s |
|                                                 Max Throughput |                 scroll |    20.03 | pages/s |
|                                        50th percentile latency |                 scroll |  952.534 |      ms |
|                                        90th percentile latency |                 scroll |  977.789 |      ms |
|                                        99th percentile latency |                 scroll |  992.309 |      ms |
|                                       100th percentile latency |                 scroll |  997.499 |      ms |
|                                   50th percentile service time |                 scroll |   952.13 |      ms |
|                                   90th percentile service time |                 scroll |  977.355 |      ms |
|                                   99th percentile service time |                 scroll |  991.834 |      ms |
|                                  100th percentile service time |                 scroll |  996.983 |      ms |
|                                                     error rate |                 scroll |        0 |       % |
|                                                 Min Throughput |             expression |      0.9 |   ops/s |
|                                              Median Throughput |             expression |     0.91 |   ops/s |
|                                                 Max Throughput |             expression |     0.91 |   ops/s |
|                                        50th percentile latency |             expression |   150678 |      ms |
|                                        90th percentile latency |             expression |   176840 |      ms |
|                                        99th percentile latency |             expression |   182169 |      ms |
|                                       100th percentile latency |             expression |   182646 |      ms |
|                                   50th percentile service time |             expression |  1125.56 |      ms |
|                                   90th percentile service time |             expression |  1286.03 |      ms |
|                                   99th percentile service time |             expression |  1303.76 |      ms |
|                                  100th percentile service time |             expression |  1332.52 |      ms |
|                                                     error rate |             expression |        0 |       % |
|                                                 Min Throughput |        painless_static |     0.94 |   ops/s |
|                                              Median Throughput |        painless_static |     0.94 |   ops/s |
|                                                 Max Throughput |        painless_static |     0.94 |   ops/s |
|                                        50th percentile latency |        painless_static |   100312 |      ms |
|                                        90th percentile latency |        painless_static |   116157 |      ms |
|                                        99th percentile latency |        painless_static |   119962 |      ms |
|                                       100th percentile latency |        painless_static |   120267 |      ms |
|                                   50th percentile service time |        painless_static |  1051.41 |      ms |
|                                   90th percentile service time |        painless_static |  1220.43 |      ms |
|                                   99th percentile service time |        painless_static |  1354.25 |      ms |
|                                  100th percentile service time |        painless_static |  1364.53 |      ms |
|                                                     error rate |        painless_static |        0 |       % |
|                                                 Min Throughput |       painless_dynamic |     0.85 |   ops/s |
|                                              Median Throughput |       painless_dynamic |     0.85 |   ops/s |
|                                                 Max Throughput |       painless_dynamic |     0.85 |   ops/s |
|                                        50th percentile latency |       painless_dynamic |   127809 |      ms |
|                                        90th percentile latency |       painless_dynamic |   147685 |      ms |
|                                        99th percentile latency |       painless_dynamic |   152236 |      ms |
|                                       100th percentile latency |       painless_dynamic |   152736 |      ms |
|                                   50th percentile service time |       painless_dynamic |  1162.78 |      ms |
|                                   90th percentile service time |       painless_dynamic |  1314.62 |      ms |
|                                   99th percentile service time |       painless_dynamic |  1485.99 |      ms |
|                                  100th percentile service time |       painless_dynamic |  1552.55 |      ms |
|                                                     error rate |       painless_dynamic |        0 |       % |
|                                                 Min Throughput |            large_terms |     0.73 |   ops/s |
|                                              Median Throughput |            large_terms |     0.73 |   ops/s |
|                                                 Max Throughput |            large_terms |     0.73 |   ops/s |
|                                        50th percentile latency |            large_terms |   176892 |      ms |
|                                        90th percentile latency |            large_terms |   204470 |      ms |
|                                        99th percentile latency |            large_terms |   210667 |      ms |
|                                       100th percentile latency |            large_terms |   211348 |      ms |
|                                   50th percentile service time |            large_terms |  1359.82 |      ms |
|                                   90th percentile service time |            large_terms |   1438.7 |      ms |
|                                   99th percentile service time |            large_terms |  1510.96 |      ms |
|                                  100th percentile service time |            large_terms |  1545.09 |      ms |
|                                                     error rate |            large_terms |        0 |       % |
|                                                 Min Throughput |   large_filtered_terms |     0.73 |   ops/s |
|                                              Median Throughput |   large_filtered_terms |     0.73 |   ops/s |
|                                                 Max Throughput |   large_filtered_terms |     0.73 |   ops/s |
|                                        50th percentile latency |   large_filtered_terms |   176867 |      ms |
|                                        90th percentile latency |   large_filtered_terms |   204577 |      ms |
|                                        99th percentile latency |   large_filtered_terms |   210807 |      ms |
|                                       100th percentile latency |   large_filtered_terms |   211565 |      ms |
|                                   50th percentile service time |   large_filtered_terms |  1358.56 |      ms |
|                                   90th percentile service time |   large_filtered_terms |  1443.66 |      ms |
|                                   99th percentile service time |   large_filtered_terms |   1483.7 |      ms |
|                                  100th percentile service time |   large_filtered_terms |  1497.43 |      ms |
|                                                     error rate |   large_filtered_terms |        0 |       % |
|                                                 Min Throughput | large_prohibited_terms |     0.75 |   ops/s |
|                                              Median Throughput | large_prohibited_terms |     0.75 |   ops/s |
|                                                 Max Throughput | large_prohibited_terms |     0.75 |   ops/s |
|                                        50th percentile latency | large_prohibited_terms |   168425 |      ms |
|                                        90th percentile latency | large_prohibited_terms |   194882 |      ms |
|                                        99th percentile latency | large_prohibited_terms |   201165 |      ms |
|                                       100th percentile latency | large_prohibited_terms |   202136 |      ms |
|                                   50th percentile service time | large_prohibited_terms |  1339.58 |      ms |
|                                   90th percentile service time | large_prohibited_terms |  1425.45 |      ms |
|                                   99th percentile service time | large_prohibited_terms |  1568.95 |      ms |
|                                  100th percentile service time | large_prohibited_terms |  1646.36 |      ms |
|                                                     error rate | large_prohibited_terms |        0 |       % |


----------------------------------
[INFO] SUCCESS (took 3850 seconds)
----------------------------------
```

#### 参考：

[https://github.com/elastic/rally](