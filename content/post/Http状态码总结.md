---
title: HTTP 状态码总结
date: 2020-07-11
layout: post
---

HTTP是一个无状态协议，但是HTTP的状态码却经常用到。



HTTP 状态码由服务器响应浏览器对服务器的请求而发出的状态信息。

HTTP 状态码有官方、非官方、扩展以及自定义等几种，相同的状态码在不同的环境下有可能有不同的含义。作为运维人员，对于常见的 HTTP 状态码肯定并不陌生且张口就来，但也有一些状态码可能职业生涯结束也不会遇见。



## 标准状态码

一个运维、开发人员我们需要熟知的状态码有哪些呢，下面会列举一些生产环境中常见的标准状态码：

### 2XX  成功

- 200 
  - OK，服务器成功返回网页
  -  Standard response for successful HTTP requests.



### 3XX  重定向

- 301 - Moved Permanently（永久跳转），请求的网页已永久跳转到新位置。

　　- This and all future requests should be directed to the given.

- 302 - Moved Temporarily（临时跳转），请求的网页已临时跳转到新位置。



### 4XX 客户端错误

- 401 - Unauthorized（未经授权），与 403 Forbidden 类似，但是专门用于需要身份验证且失败或尚未提供的情况。

- 403 - Forbidden（禁止访问），服务器拒绝请求
  - forbidden request (matches a deny filter) => HTTP 403
    - The request was a legal request, but the server is refusing to respond to it.
    - 403表示服务器端拒绝接受客户端发送过来的请求，而且一般不会给出提示原因，为何给予拒绝。不过一般会是因为用户无权限访问造成的。在我工作过程中，经常会遇到403的问题，因为我们对接口的权限管理很严格，如果新增的接口没有正确配置权限，就会造成403的问题。

- 404 - Not Found, 服务器找不到请求的页面。
    - The requested resource could not be found but may be available again in the future.
    - 404可能是所有程序员最熟悉的状态码了吧，无需过多描述，就是请求的资源在服务器端不存在，一般为请求的URL不对或者资源不存在。

- 499 客户端主动断开连接
  - Client Closed Request ， 客户端主动断开连接。
  - 499 是由于超过客户端设置的请求超时时间，客户端主动关闭连接，服务器 code 为 499
  - 是指一次http请求在客户端指定的时间内没有返回响应，此时，客户端会主动断开连接，此时表象为客户端无响应返回，而 nginx 的日志中会 status code 为 499 。



### 5XX 服务器错误

- 500 - Internal Server Error（内部服务器错误）
  -  internal error in haproxy => HTTP 500
    - A generic error message, given when no more specific message is suitable.
    - 500状态码表示的是服务器内部执行异常，一般都表现为程序上的bug，例如代码在执行过程中抛出异常，例如常见的空指针。

- 502 - Bad Gateway（坏的网关）, 一般是网关服务器请求后端服务时，后端服务没有按照 http 协议正确返回结果。

  - the server returned an invalid or incomplete response => HTTP 502

    - The server was acting as a gateway or proxy and received an invalid response from the upstream server.

    - 502状态码一般会展现bad gateway错误网关类型的信息。

      主要是由于客户端向服务器端请求超时，比如在服务器端网络状况不好的情况下，同时又有多个客户端向服务器端发送请求，会造成服务器端资源不够，无法正常响应，便会返回这个结果。

      一般最简单的解决方就是刷新的方式，有很多由于有缓存的情况，直接从本地拿数据，就不会再报502错误。

- 503 - Service Unavailable（服务当前不可用）, 可能因为超载或停机维护。
  - no server was available to handle the request => HTTP 503
    - The server is currently unavailable (because it is overloaded or down for maintenance).
    - 503状态码表示服务器无法处理请求，一般表现为服务器宕机或者处于超负荷状态。不过这一般都是暂时性的情况，在服务重启或者负载均衡处理后，服务会继续处于可用状态。

- 504 - Gateway Timeout（网关超时）, 一般是网关服务器请求后端服务时，后端服务没有在特定的时间内完成服务。

  -  the server failed to reply in time => HTTP 504
    - The server was acting as a gateway or proxy and did not receive a timely response from the upstream server.	
    - 504状态码一般网关在转发过程中，超过设定的时间仍未收到上游服务器的响应。

  - 505（HTTP 版本不受支持）服务器不支持请求中所用的 HTTP 协议版本。



对于生产环境中常用的状态码我们需要非常熟悉，这些状态码会经常用于判断服务的可用性上，也很方便的适用于前后端联调时出错的判断。