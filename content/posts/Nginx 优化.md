---
title: Nginx 优化总结
date: 2018-04-21
layout: post
---

### **优化 Nginx 进程数量**

```
worker_processes 1; # 指定 Nginx 要开启的进程数，结尾的数字就是进程的个数，可以为 auto
```

这个参数调整的是 Nginx 服务的 worker 进程数，Nginx 有 Master 进程和 worker 进程之分，Master 为管理进程、真正接待“顾客”的是 worker 进程。

进程个数的策略：worker 进程数可以设置为等于 CPU 的核数。高流量高并发场合也可以考虑将进程数提高至 CPU 核数 x 2。这个参数除了要和 CPU 核数匹配之外，也与硬盘存储的数据及系统的负载有关，设置为 CPU 核数是个好的起始配置，也是官方建议的。

当然，如果想省麻烦也可以配置为worker_processes auto;，将由 Nginx 自行决定 worker 数量。当访问量快速增加时，Nginx 就会临时 fork 新进程来缩短系统的瞬时开销和降低服务的时间。



#### **Nginx 事件处理模型优化**

Nginx 的连接处理机制在不同的操作系统中会采用不同的 I/O 模型，在 linux 下，Nginx 使用 epoll 的 I/O 多路复用模型，在 Freebsd 中使用 kqueue 的 I/O 多路复用模型，在 Solaris 中使用 /dev/poll 方式的 I/O 多路复用模型，在 Windows 中使用 icop，等等。

配置如下：

```
events {
  use epoll;
}
```



#### **单个进程允许的客户端最大连接数**

通过调整控制连接数的参数来调整 Nginx 单个进程允许的客户端最大连接数。

```
events {
  worker_connections 20480;
}
```

worker_connections 也是个事件模块指令，用于定义 Nginx 每个进程的最大连接数，默认是 1024。

最大连接数的计算公式如下：

max*clients = worker*processes * worker_connections;

如果作为反向代理，因为浏览器默认会开启 2 个连接到 server，而且 Nginx 还会使用fds（file descriptor）从同一个连接池建立连接到 upstream 后端。则最大连接数的计算公式如下：

max*clients = worker*processes * worker_connections / 4;

另外，进程的最大连接数受 Linux 系统进程的最大打开文件数限制，在执行操作系统命令 ulimit -HSn 65535或配置相应文件后， worker_connections 的设置才能生效。

## #### **配置获取更多连接数**

默认情况下，Nginx 进程只会在一个时刻接收一个新的连接，我们可以配置multi_accept 为 on，实现在一个时刻内可以接收多个新的连接，提高处理效率。该参数默认是 off，建议开启。

```
events {
  multi_accept on;
}
```



## **配置 worker 进程的最大打开文件数**

调整配置 Nginx worker 进程的最大打开文件数，这个控制连接数的参数为 worker*rlimit*nofile。该参数的实际配置如下:

```
worker_rlimit_nofile 65535;
```

可设置为系统优化后的 ulimit -HSn 的结果

## #### TCP 优化

```
http {
  sendfile on;
  tcp_nopush on;
  keepalive_timeout 120;
  tcp_nodelay on;
}
```



第一行的 sendfile 配置可以提高 Nginx 静态资源托管效率。sendfile 是一个系统调用，直接在内核空间完成文件发送，不需要先 read 再 write，没有上下文切换开销。

TCP*NOPUSH 是 FreeBSD 的一个 socket 选项，对应 Linux 的 TCP*CORK，Nginx 里统一用 tcp_nopush 来控制它，并且只有在启用了 sendfile 之后才生效。启用它之后，数据包会累计到一定大小之后才会发送，减小了额外开销，提高网络效率。

TCP*NODELAY 也是一个 socket 选项，启用后会禁用 Nagle 算法，尽快发送数据，某些情况下可以节约 200ms（Nagle 算法原理是：在发出去的数据还未被确认之前，新生成的小数据先存起来，凑满一个 MSS 或者等到收到确认后再发送）。Nginx 只会针对处于 keep-alive 状态的 TCP 连接才会启用 tcp*nodelay。

#### **优化连接参数**

```
http {
  client_header_buffer_size 32k;
  large_client_header_buffers 4 32k;
  client_max_body_size 1024m;
  client_body_buffer_size 10m;
}
```



这部分更多是更具业务场景来决定的。例如client*max*body_size用来决定请求体的大小，用来限制上传文件的大小。上面列出的参数可以作为起始参数。

## #### **配置压缩优化**

1、Gzip 压缩

我们在上线前，代码（JS、CSS 和 HTML）会做压缩，图片也会做压缩（PNGOUT、Pngcrush、JpegOptim、Gifsicle 等）。对于文本文件，在服务端发送响应之前进行 GZip 压缩也很重要，通常压缩后的文本大小会减小到原来的 1/4 - 1/3。

```
http {
  gzip on;
  gzip_buffers 16 8k;
  gzip_comp_level 6;
  gzip_http_version 1.0;
  gzip_min_length 1000;
  gzip_proxied any;
  gzip_vary on;
  gzip_types
    text/xml application/xml application/atom+xml application/rss+xml application/xhtml+xml image/svg+xml
    text/javascript application/javascript application/x-javascript
    text/x-json application/json application/x-web-app-manifest+json
    text/css text/plain text/x-component
    font/opentype application/x-font-ttf application/vnd.ms-fontobject
    image/x-icon;
  gzip_disable "MSIE [1-6]\.(?!.*SV1)";
}
```

这部分内容比较简单，只有两个地方需要解释下：

gzip_vary 用来输出 Vary 响应头，用来解决某些缓存服务的一个问题，详情请看我之前的博客：HTTP 协议中 Vary 的一些研究。

gzip_disable 指令接受一个正则表达式，当请求头中的 UserAgent 字段满足这个正则时，响应不会启用 GZip，这是为了解决在某些浏览器启用 GZip 带来的问题。

默认 Nginx 只会针对 HTTP/1.1 及以上的请求才会启用 GZip，因为部分早期的 HTTP/1.0 客户端在处理 GZip 时有 Bug。现在基本上可以忽略这种情况，于是可以指定 gzip*http*version 1.0 来针对 HTTP/1.0 及以上的请求开启 GZip。



**静态资源优化**

静态资源优化，可以减少连接请求数，同时也不需要对这些资源请求打印日志。但副作用是资源更新可能无法及时。

```
server {
    # 图片、视频
    location ~ .*\.(gif|jpg|jpeg|png|bmp|swf|flv|mp4|ico)$ {
      expires 30d;
      access_log off;
    }
    # 字体
    location ~ .*\.(eot|ttf|otf|woff|svg)$ {
      expires 30d;
      access_log off;
    }
    # js、css
    location ~ .*\.(js|css)?$ {
      expires 7d;
      access_log off;
    }
}
```





附一份Nginx 配置文件

```
user	deploy;
worker_processes	4;

error_log  /var/log/nginx/error.log warn;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

pid	/var/run/nginx.pid;
worker_rlimit_nofile 65535;

events {
    use epoll;
    worker_connections  65535;
}

http {
#    lua_shared_dict limit 10m;
#    access_by_lua_file /etc/nginx/lua/wxfe.lua;
    include	mime.types;
    default_type	application/octet-stream;
    charset utf-8;
    map $upstream_response_time $upstream_response_timer {
        default $upstream_response_time;
           ""        0;
           "-"        0;
    }
    map $status $level {
        default debug;
        ~^[2] info;
        ~^[3] info;
        ~^[4] warn;
        ~^[5] error;
 }
 #log_format	main  '$proxy_add_x_forwarded_for||$remote_user||$time_local||$request||$status||$body_bytes_sent||$http_referer||$http_user_agent||$remote_addr||$http_host||$request_body||$upstream_addr||$request_time||$upstream_response_time';
    log_format main  escape=json '{ "time_local": "$time_local", '
                         '"remote_user": "$remote_user", '
                         '"remote_addr": "$remote_addr", '
                         '"http_referer": "$http_referer", '
                         '"request": "$request", '
                         '"method": "$request_method", '
                         '"url_path": "$request_uri", '
                         '"request_body": "$request_body", '
                         '"status": $status, '
			 '"level": "$level",'
                         '"body_bytes_sent": $body_bytes_sent, '
                         '"http_referer": "$http_referer", '
                         '"http_user_agent": "$http_user_agent", '
                         '"http_host": "$http_host", '
                         '"business": "ngx_access-$http_host", '
                         '"http_x_forwarded_for": "$http_x_forwarded_for", '
                         '"upstream_addr": "$upstream_addr",'
                         '"upstream_response_time": "$upstream_response_timer",'
                         '"request_time": $request_time'
                         ' }';
    keepalive_timeout 65;
    tcp_nodelay on;
    server_tokens off;
    large_client_header_buffers 4 256k;
    client_max_body_size 300m;
    client_body_buffer_size 16m;
    server_names_hash_bucket_size 64;
    access_log  /var/log/nginx/access.log  main;

    sendfile	on;

    gzip	on;
    gzip_min_length 1k;
    gzip_buffers 16 64k;
    gzip_http_version 1.1;
    gzip_comp_level 6;
    gzip_types text/plain application/x-javascript text/css application/xml;
    gzip_vary on;

    include conf.d/*.conf;
}
```

