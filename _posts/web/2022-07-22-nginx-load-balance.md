---
layout:     post
title:      Nginx 负载均衡配置
subtitle:   "Configure Loading Balance in Nginx"
date:       2022-07-22 19:16:00
author:     "Zewan"
catalog:    true
mathjax:    true
header-style: text
target: _blank
tags:
    - Web
    - Nginx
    - 部署
---

负载均衡可以理解为一种平衡策略，用于管理和分发客户端的请求，将其交由各服务器节点处理。该策略前提是有多个节点同时工作，以减轻单节点的压力，提高性能和可靠性。即使有一个节点不可用，也有其他节点可使用，从而提高系统的健壮性。（节点可以是集群中的**服务器节点**或单机器中的 **Docker 服务节点**）

实现负载均衡的方式有很多，例如 LVS、Nginx 等，本文主要介绍 Nginx 负载均衡及其配置方式。

> 友情提示：理解本文需要读者了解过 Nginx 基础配置，成功部署过 Nginx 应用的话会很好理解

## Nginx 负载均衡配置基础语法

Nginx 负载均衡在 Nginx Server 节点配置文件中设置，在 Ubuntu 系统中通常位于 `/etc/nginx/conf.d/`，可在该目录下新建 `*.conf` 文件编写节点配置信息。

下面代码配置 192.168.159.1 和 192.168.159.2 节点作为负载均衡的流量节点，也就是说，当用户请求 `http://aaa.xyz` 时，Nginx 会通过负载均衡策略分发至这两个节点其中之一，此处策略为随机分配。

```conf
upstream cluster {
    server 192.168.159.1:80;
    server 192.168.159.2:80;
}
server {
    listen 80;
    server_name aaa.xyz;
    location / {
        proxy_pass http://cluster;
    }
}
```

## 负载均衡策略

### 轮询

Nginx 负载均衡策略默认为轮询方式，上面的代码即为该方式，请求会随机派发到配置的服务节点上。

### 权重分配

配置示例：

```conf
upstream cluster {
    server 192.168.159.1:80 weight=4 max_fails=5 fail_timeout=300s;
    server 192.168.159.2:80 weight=3 max_fails=5 fail_timeout=300s;
}
```

- **weight**: 权重，值越高则请求被派发至该节点的概率越高，可根据服务器性能配置来设置
- **max_fails**: 请求失败多少次将踢出可分发节点队列
- **fail_timeout**: 在达到 `max_fails` 的次数踢出队列后重新尝试探测请求的时间

### 哈希分配

在 Web 应用中，若使用 Session 机制记录用户登录信息，且使用了负载均衡，那么登录请求的服务节点和后续请求的节点可能不一致，从而导致登录信息丢失。例如，用户的登录请求在某个服务节点上通过了，当用户进行其他请求时，由于负载均衡机制，请求可能被分发给集群中的某一个，那么已登录某节点的用户重新定位到另一个节点时，登录信息可能会丢失，这样的方式肯定是不可取的。

> 若是使用 Token 机制记录登录信息，则不会存在该问题，因为该机制后台服务节点不记录信息

可以采用 ip_hash 的方式解决这个问题，如果客户已经访问了某节点，当用户再次访问时，会将该请求通过哈希算法，自动定位到该节点。

```conf
upstream cluster {
    ip_hash;
    server 192.168.159.1:80;
    server 192.168.159.2:80;
}
```

哈希分配策略根据客户端 IP 来分配节点，例如第一次访问请求被派发给了 192.168.159.1:80，那么后续请求都会发送至该节点，以此解决 Session 共享的问题。

### 最少连接分配

在配置的节点列表中，判断哪一节点分的连接最少，则将请求分给该节点。

```conf
upstream cluster {
    least_conn;
    server 192.168.159.1:80;
    server 192.168.159.2:80;
}
```

## 完整示例

在 `/etc/nginx/conf.d/` 目录下新建 `test.conf` 和 `testapi.conf` 两个文件，分别将前后端节点部署至 `http://test.com` 和 `http://api.test.com` 两个域名下。

前端服务使用 Docker 在同一机器 159.1 中建立三个节点，分别为 192.168.159.1:8081 ~ 8083。

`test.conf` 内容：

```conf
upstream test.cluster {
    server 192.168.159.1:8081;
    server 192.168.159.1:8082;
    server 192.168.159.1:8083;
}
server {
    listen 80;
    server_name test.com;
    location / {
        proxy_pass http://test.cluster;
    }

    access_log /var/log/nginx/test.com.access.log;
    error_log /var/log/nginx/test.com.error.log;
}
```

后端服务使用 Docker 在 159.1 建立两个节点 8001 ~ 8002，在 159.2 建立三个节点 8001 ~ 8003。

`testapi.conf` 内容：

```conf
upstream api.test.cluster {
    server 192.168.159.1:8001 weight=1 max_fails=5 fail_timeout=300s;
    server 192.168.159.1:8002 weight=1 max_fails=5 fail_timeout=300s;
    server 192.168.159.2:8001 weight=2 max_fails=5 fail_timeout=300s;
    server 192.168.159.2:8002 weight=2 max_fails=5 fail_timeout=300s;
    server 192.168.159.2:8003 weight=2 max_fails=5 fail_timeout=300s;
}
server {
    listen 80;
    server_name api.test.com;
    location / {
        proxy_pass http://api.test.cluster;
    }

    access_log /var/log/nginx/api.test.com.access.log;
    error_log /var/log/nginx/api.test.com.error.log;
}
```