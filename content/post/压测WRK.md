---
title: 压测工具之WRK
date: 2020-05-08
layout: post
---

WRK 基于事件机制的高性能 HTTP 压力测试工具。

WRK 也是比较常见的压测工具，其他的还有  AB，JMeter

wrk 负载测试时可以运行在一个或者多核CPU，wrk 结合了可伸缩的事件通知系统 epoll 和 kqueue 等多线程设计思想。wrk 不仅能测试单条 URL，还能通过`LuaJIT`脚本实现对不同的 URL 和参数、请求内容进行测试。

[wrk - a HTTP benchmarking tool](https://github.com/wg/wrk)



### 安装

- CentOS

```
sudo yum groupinstall 'Development Tools'
sudo yum install -y openssl-devel git 
git clone https://github.com/wg/wrk.git wrk
cd wrk
make
# move the executable to somewhere in your PATH
sudo cp wrk /usr/bin/
```

- Mac 

```
brew install wrk
```



### 参数

```
 wrk <选项> <被测HTTP服务的URL>                            
  Options:                                            
    -c, --connections <N>  跟服务器建立并保持的TCP连接数量  
    -d, --duration    <T>  压测时间           
    -t, --threads     <N>  使用多少个线程进行压测   

    -s, --script      <S>  指定Lua脚本路径       
    -H, --header      <H>  为每一个HTTP请求添加HTTP头      
        --latency          在压测结束后，打印延迟统计信息   
        --timeout     <T>  超时时间     
    -v, --version          打印正在使用的wrk的详细版本信息

  <N>代表数字参数，支持国际单位 (1k, 1M, 1G)
  <T>代表时间参数，支持时间单位 (2s, 2m, 2h)
```



#### 普通示例

```
# 启用1个线程，每个线程并发100个连接，压测10秒
$ wrk -t1 -c100 -d10s --latency http://test.com
Running 10s test @ http://test.com（压测时间30s）
  1 threads and 100 connections（共1个测试线程，100个连接）
  Thread Stats   Avg      Stdev     Max   +/- Stdev
            （平均值）    （标准差） （最大值）（正负一个标准差所占比例）
    Latency   316.94ms  319.23ms   1.99s    89.96%
    （延迟）
    Req/Sec   261.78    203.48     1.00k    77.66%
   （每秒处理中的请求数）
  Latency Distribution
       （延迟分布）
     50%  302.71ms
     75%  378.96ms
     90%  630.04ms
     99%    1.60s
  2634 requests in 10.05s, 5.44MB read
  （10.05秒内共处理完成了2634个请求，读取了5.44MB数据）
  Socket errors: connect 0, read 0, write 0, timeout 44
  （Socket错误数统计，0个连接错误，0个读取错误，0个写入错误，44个超时）
Requests/sec:    262.22（平均每秒262.22个请求）
Transfer/sec:    554.27KB（平均每秒读取数据554.27KB）
```



还可以使用 Lua 脚本 实现个性化压测 

```
# -s 指定 lua 脚本
wrk -t10 -c1000 -d40s --latency -s test.lua http://test.com
```

其中post.lua  

```
-- example HTTP POST script which demonstrates setting the
-- HTTP method, body, and adding a header

wrk.method = "POST"
wrk.body   = "foo=bar&baz=quux"
wrk.headers["Content-Type"] = "application/x-www-form-urlencoded"
```

