---
title: Nginx实战
date: 2018-02-03
tags: Nginx
---

# 搭建一个简易的代理服务
nginx 常常用来作为代理服务器，这代表着服务器接收请求，然后将它们传递给被代理服务器，得到请求的响应，再将它们发送给客户端。 我们将配置一个基本的代理服务器，它会处理本地图片文件的请求并返回其他的请求给被代理的服务器。在这个例子中，两个服务器都会定义在一个 nginx 实例中。 首先，通过在 nginx 配置文件中添加另一个 server 区块，来定义一个被代理的服务器，像下面的配置：
```
server {
    listen 8080;
    root /data/up1;

    location / {
    }
}
```
<!-- more --> 


上面就是一个简单的服务器，它监听在 8080 端口（之前，listen 并没被定义，是因为默认监听的 80 端口）并且会映射所有的请求给 本地文件目录 /data/up1。创建该目录，然后添加 index.html 文件。注意，root 指令是放在 server 上下文中。当响应请求的 location 区块中，没有自己的 root 指令，上述的 root 指令才会被使用。 接着，使用前面章节中的 server 配置，然后将它改为一个代理服务配置。在第一个 location 区块中，放置已经添加被代理服务器的协议，名字和端口等参数的 proxy_pass 指令（在这里，就是 http://localhost:8080）:
```
server {
    location / {
        proxy_pass http://localhost:8080;
    }

    location /images/ {
        root /data;
    }
}
```
## location相关

我们将修改第二个 location 区块，使他返回一些典型后缀的图片文件请求，
现在它只会映射带有 /images/ 前缀的请求到 /data/images 目录下。
修改后的 location 指令如下：

```
location ~ \.(gif|jpg|png)$ {
    root /data/images;
}
```
该参数是一个正则表达式，它会匹配所有以 .gif，.jpg 或者 .png 结尾的 URIs。
一个正则表达式需要以 ~ 开头。匹配到的请求会被映射到 /data/images 目录下。
 当 nginx 在选择 location 去响应一个请求时，它会先检测带有前缀的 location 指令，记住先是检测带有最长前缀的 location，然后检测正则表达式。
 如果有一个正则的匹配的规则，nginx 会选择该 location，否则，会选择之前缓存的规则。 最终，一个代理服务器的配置结果如下：
```
 server {
    location / {
        proxy_pass http://localhost:8080/;
    }

    location ~ \.(gif|jpg|png)$ {
        root /data/images;
    }
}
```

该服务器会选择以 .gif，.jpg，或者 .png 结束的请求并且映射到 /data/images 目录（通过添加 URI 给 root 指令的参数），
接着将其他所有的请求映射到上述被代理的服务器。 为了使用新的配置，像前几个章节描述的一样，需要向 nginx 发送重载信号。
 这还有很多其他的指令，可以用于进一步配置代理连接。



  ### location 配置
 1. = 开头表示精确匹配
 2. ^~ 开头表示uri以某个常规字符串开头，不是正则匹配
 3. ~ 开头表示区分大小写的正则匹配;
 4. ~* 开头表示不区分大小写的正则匹配
 5. / 通用匹配, 如果没有其它匹配,任何请求都会匹配到
