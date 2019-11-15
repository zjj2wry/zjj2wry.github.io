---
title: "golang 大量 TIME_WAIT"
date: 2019-11-14T20:29:58+08:00
lastmod: 2019-11-14T20:29:58+08:00
---

- [TIME_WAIT and its design implications for protocols and scalable client server systems](http://www.serverframework.com/asynchronousevents/2011/01/time-wait-and-its-design-implications-for-protocols-and-scalable-servers.html)
- [tcp rfc](https://tools.ietf.org/html/rfc793#page-37)
- [tuning-the-go-http-client-library-for-load-testing](http://tleyden.github.io/blog/2016/11/21/tuning-the-go-http-client-library-for-load-testing/)<!--more-->

## 前提

![TCP-termination](../../images/TCP-termination.png)
TIME_WAIT 状态会出现在 tcp 四次挥手主动关闭连接的一方。

    TIME-WAIT - represents waiting for enough time to pass to be sure
    the remote TCP received the acknowledgment of its connection
    termination request.

TIME_WAIT 会等待 2 msl 再关闭连接(rfc 中定义的 2分钟，linux 中是 30s)

1. 如果没有等待两秒直接关闭连接，老的报文出现延迟重传，新建立的连接五元组相同，导致异常
2. 进入 TIME_WAIT 后主动关闭连接的一方需要确认对方能收到 ack，如果没有收到 ack，被动关闭连接的一方会重传。假设没有等待直接关闭连接，重传会接受到 reset，导致异常

## golang TIME_WAIT

golang 导致大量 TIME_WAIT 的主要原因是频繁的建立短链接。

### 没有关闭或读取 response body


    //https://golang.org/src/net/http/response.go
    // Body represents the response body.
    //
    // The response body is streamed on demand as the Body field
    // is read. If the network connection fails or the server
    // terminates the response, Body.Read calls return an error.
    //
    // The http Client and Transport guarantee that Body is always
    // non-nil, even on responses without a body or responses with
    // a zero-length body. It is the caller's responsibility to
    // close Body. The default HTTP client's Transport may not
    // reuse HTTP/1.x "keep-alive" TCP connections if the Body is
    // not read to completion and closed.
    //
    // The Body is automatically dechunked if the server replied
    // with a "chunked" Transfer-Encoding.
    //
    // As of Go 1.12, the Body will also implement io.Writer
    // on a successful "101 Switching Protocols" response,
    // as used by WebSockets and HTTP/2's "h2c" mode.
    Body io.ReadCloser

如下，会有大量的 time wait 出现在客户端
![not-close-or-read-resp-body](../../images/not-close-or-read-resp-body.jpg)

抓包查看发现有大量的 FIN-ACK(server 被动关闭连接，server 端会直接进入到 close wait 状态)
![not-close-or-read-resp-body-cap](../../images/not-close-or-read-resp-body-cap.jpg)

### 使用了 golang 默认的 RoundTripper

如果没有设置 MaxIdleConnsPerHost，其他的都会被回收，也会导致客户端大量 time wait, 抓包后效果和上面的一致。另外就是 ss 命令比 netstat 快很多很多

上面两个原因都是因为客户端没有复用连接导致的，因为 source port 是自动分配的，有范围限制，短链接需要回收端口，不然会导致无法新建连接。如果你是 server 端，会因为无法新建连接导致性能问题。在 k8s 环境里，通过 svc 访问，大量的短链接会导致 coredns 的 qps 较高。

## 常用命令
查看端口访问
```sh
cat /proc/sys/net/ipv4/ip_local_port_range
```

```
netstat
    -a --all 显示所有
    -n --numeric 显示数字
    -p 显示进程相关
    -t 显示 tcp
    -u 显示 udp
    -s 统计信息
    -l --lisening 显示监听中的
```

查看统计信息
```bash
netstat -s
```

查看所有的 tcp 连接
```bash
netstat -ant
```

查看路由表
```bash
netstat -r
```
