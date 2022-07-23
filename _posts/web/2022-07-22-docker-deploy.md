---
layout:     post
title:      Docker 分离部署前后端节点（Vue&Django为例）
subtitle:   "Deploy Frontend and Backend Nodes Separately with Docker"
date:       2022-07-22 10:16:00
author:     "Zewan"
catalog:    true
mathjax:    true
header-style: text
target: _blank
tags:
    - Web
    - 部署
    - Nginx
    - Docker
    - Vue
    - Django
---

在[『服务器部署 Vue 和 Django 项目的全记录』](/2021/06/25/deploy-django-vue/)一文中，介绍了在服务器中使用 Nginx 部署前后端项目的过程。然而，当 Web 应用流量增多时，需要考虑负载均衡、流量分发、容灾等情况，原生的部署方式通常难以满足需求。此时，引入 Docker 部署多节点，能够在单台高性能服务器或服务器集群中搭建更完善的部署架构。

本文主要以 Vue 和 Django 项目为例介绍 Docker 部署的流程，稍带 Docker 简介和基础的 Nginx 负载均衡配置。

## Docker 简介与安装

> 简单介绍 Docker 相关概念，具体需要读者另外学一学，推荐[『Docker-从入门到实践』](https://yeasy.gitbook.io/docker_practice/)

### Docker 是什么

Docker 是一个开源的应用容器引擎，可以让开发者打包应用和依赖到一个轻量级、可移植的容器中，并发布到任何流行的 Linux 机器上，也可以实现虚拟化。容器使用沙箱机制，相互之间不存在接口，其与宿主机通过端口转发进行通信，性能开销低。

Docker 部署 Web 应用有以下优点：

- 容器适合持续集成和持续交付（CI/CD）流程
- 响应式部署和扩展，其可移植性和轻量级的特性支持实时扩展或拆除服务
- Docker 轻巧快速，支持开发者在同一机器上运行更多工作负载

### Docker 工作流程

Docker 包括三个概念：

- **镜像（Image）**：相当于一个 root 文件系统。
- **容器（Container）**：镜像和容器的关系类似于对象程序设计中的类和实例，镜像是静态的定义，容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等。
- **仓库（Repository）**：仓库可看成一个代码控制中心，用来保存镜像。

Docker 的工作流程通常为：

1. 从仓库中拉取（pull）官方或基准镜像
2. 在 Dockerfile 中描述应用和安装依赖的指令，构建镜像
3. 由镜像创建和运行容器

### Docker 安装

见[『Ubuntu 安装 Docker 环境』](/2022/07/21/install-docker/).

## 部署架构

**在不考虑多节点负载均衡时**，本文的部署架构如下：

前后端项目分离部署，分别部署在两个 Nginx 节点，对应两个域名或两个端口。

<img alt="deploy-structure" src="/img/in-post/post-docker-deploy/deploy-structure.png" style="width: 60%">

## Nginx + Docker 部署前端节点

首先，Vue 项目打包为 dist 文件夹，同目录下新建 `Dockerfile`、`vhosts.conf` 文件和 `logs` 文件夹，作用见下列代码块中的注释。

```bash
.
├── dist            # Vue 项目打包用以部署的文件夹
├── Dockerfile      # 用于建立 Docker 镜像
├── vhosts.conf     # 容器中启动 Nginx 服务的配置文件
└── logs            # 映射容器中的 Nginx 日志目录，以便在宿主机查看日志
```

`Dockerfile` 文件内容：

```Dockerfile
# 设置基础镜像
FROM nginx:latest
#设置CTS时区
RUN cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo 'Asia/Shanghai' > /etc/timezone
# 将dist文件中的内容复制到 /usr/share/nginx/html/ 这个目录下面
COPY ./dist /usr/share/nginx/html/
#用本地的 vhosts.conf 配置来替换 nginx 镜像里的默认配置
COPY vhosts.conf /etc/nginx/conf.d/vhosts.conf
```

`vhosts.conf` 文件内容：

```conf
server {
    listen       80;
    server_name  localhost;
    access_log  /var/log/nginx/host.access.log  main;
    error_log  /var/log/nginx/error.log  error;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        try_files $uri $uri/ /index.html;
    }
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

### 构建 Docker 镜像

在终端中输入以下命令，根据 `Dockerfile` 构建镜像。

```bash
docker build -f Dockerfile -t nginx/hsheng-mall:v1.0.0 .
# 镜像名称：nginx/hsheng-mall
# 版本号：v1.0.0
```

### 运行 Docker 容器

依据创建的镜像 `nginx/hsheng-mall:v1.0.0` 运行容器。

```bash
docker run -d -p 8081:80 --name=hsheng-mall -v /home/hsheng/www/hsheng-mall/logs:/var/log/nginx nginx/hsheng-mall:v1.0.0
# 宿主机 8081 端口映射容器 80 端口
# 容器名称：hsheng-mall
# 宿主机 /home/hsheng/www/hsheng-mall/logs 目录映射容器 /var/log/nginx 目录
# 镜像名称：nginx/hsheng-mall:v1.0.0
```

### 宿主机 Nginx 转发

与原生 Nginx 部署类似，在 `/etc/nginx/conf.d` 目录下创建配置文件 `hsheng-mall.conf`，内容如下：

```conf
server {
    listen 443 ssl; # 端口，若部署 https 域名则为 443
    server_name aaa.abc.com; # 域名或 IP

    location / {
        proxy_pass http://127.0.0.1:8081;   # 转发本机（宿主机） 8081 端口，已与 Docker 端口建立映射
        proxy_redirect default;
    }

    ssl_certificate   /home/hsheng/www/hsheng-mall/ssl_certs/aaa.abc.com_bundle.crt;    # ssl证书绝对路径
    ssl_certificate_key  /home/hsheng/www/hsheng-mall/ssl_certs/aaa.abc.com.key;    # ssl证书私钥绝对路径
    ssl_session_timeout 5m;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
    ssl_prefer_server_ciphers on;
}

server {
    listen 80;
    server_name aaa.abc.com;
    # 把http的域名请求转成https
    return 301 https://$host$request_uri;
}
```

而后重启 Nginx 服务，即可通过 `server_name` 访问 Docker 的应用服务。

```bash
sudo nginx -s reload
```

## Docker 部署 uWSGI 后端节点

项目部署目录结构如下：

```bash
.
├── src                     # Django 项目源码
│   ├── manage.py
│   ├── requirements.txt    # Python 项目依赖包
│   ├── uwsgi.ini           # uWSGI 配置文件
│   ├── start.sh            # Django 服务启动脚本
|   └── ...
├── Dockerfile              # 用于建立 Docker 镜像
└── logs                    # 映射容器中的 uWSGI 日志目录，以便在宿主机查看日志
```

`Dockerfile` 文件内容：

```Dockerfile
FROM python:3.8
RUN cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo 'Asia/Shanghai' > /etc/timezone
RUN mkdir -p /var/www/html/backend
COPY ./src /var/www/html/backend/
WORKDIR /var/www/html/backend

RUN pip install -i https://pypi.doubanio.com/simple uwsgi
RUN pip install -i https://pypi.doubanio.com/simple/ -r requirements.txt

# Windows环境下编写的start.sh每行命令结尾有多余的\r字符，需移除
RUN sed -i 's/\r//' ./start.sh
RUN chmod +x ./start.sh
```

`uwsgi.ini` 文件内容：

```ini
[uwsgi]
socket = 0.0.0.0:8000   # 容器内 uWSGI 服务须为 0.0.0.0，以便与宿主机建立正常连接
project = backend
base = /var/www/html
base-app = hshengmall
chdir = %(base)/%(project)
wsgi-file = %(base)/%(project)/%(base-app)/wsgi.py
master = true
processes = 8
threads = 4
enable-threads = true
buffer-size = 65536
post-buffering = 32768
vacuum = true
pidfile = %(base)/uwsgi/%(project)-master.pid
daemonize = %(base)/uwsgi/uwsgi.log
chmod-socket = 664
# 设置一个请求的超时时间(秒)，如果一个请求超过了这个时间，则请求被丢弃
harakiri = 300
# 当一个请求被harakiri杀掉会，会输出一条日志
harakiri-verbose = true
```

`start.sh` 文件内容：

```bash
python manage.py makemigrations&&
python manage.py migrate&&
uwsgi --ini /var/www/html/backend/uwsgi.ini
```

### 构建 Docker 镜像

在终端中输入以下命令，根据 `Dockerfile` 构建镜像。

```bash
docker build -f Dockerfile -t python/hsheng-mall-backend:v1.0.0 .
```

### 运行 Docker 容器

依据创建的镜像 `python/hsheng-mall-backend:v1.0.0` 运行容器。

```bash
docker run -it -p 8001:8000 --name=hsheng-mall-backend -v /home/hsheng/www/hsheng-mall-backend/logs:/var/www/html/uwsgi -d python/hsheng-mall-backend:v1.0.0
```

### 启动服务

进入容器：

```bash
docker exec -it <container_id> /bin/bash
```

运行启动脚本：

```bash
./start.sh
```

这样即成功启动了一个后端服务容器，若想做多节点负载均衡，可以修改端口映射关系，按上述步骤多创建和启动几个容器。

### 宿主机 Nginx 转发

在 `/etc/nginx/conf.d` 目录下创建配置文件 `hsheng-mall-backend.conf`，内容如下：

```conf
server {
    listen 443 ssl;
    server_name api-aaa.abc.com;

    location / {
        include /etc/nginx/uwsgi_params;
        uwsgi_pass 127.0.0.1:8001;  # 转发本机（宿主机） 8001 端口，已与 Docker 端口建立映射
    }

    ssl_certificate   /home/hsheng/www/hsheng-mall-backend/ssl_certs/api-aaa.abc.com_bundle.crt;
    ssl_certificate_key  /home/hsheng/www/hsheng-mall-backend/ssl_certs/api-aaa.abc.com.key;
    ssl_session_timeout 5m;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
    ssl_prefer_server_ciphers on;
}

server {
    listen 80;
    server_name api-aaa.abc.com;
    return 301 https://$host$request_uri;
}
```

而后重启 Nginx 服务，即可通过 `server_name` 请求后端服务。

```bash
sudo nginx -s reload
```

## *Nginx 负载均衡

简略描述一个负载均衡的结构：Nginx + Docker 多节点部署架构。

前端节点为静态节点，通常只需要单个节点即可，可使用 CDN 加速优化访问。因此，当请求流量大时，主要通过增多后端 Docker+uWSGI 节点进行负载均衡。

<img alt="deploy-structure-total" src="/img/in-post/post-docker-deploy/deploy-structure-total.png" style="width: 80%">

为实现上图架构，首先根据本文第四章节[「Docker 部署 uWSGI 后端节点」](/2022/07/22/docker-deploy/#docker-%E9%83%A8%E7%BD%B2-uwsgi-%E5%90%8E%E7%AB%AF%E8%8A%82%E7%82%B9)创建和启动 3 个 Docker 后端服务节点，分别映射至宿主机 8001 ~ 8003 端口。

而后，修改宿主机用于部署后端的 Nginx 配置文件，例如本文的 `hsheng-mall-backend.conf`，添加 `upstream`。修改后文件内容应为：

```conf
upstream uwsgicluster {
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
    server 127.0.0.1:8003;
}
server {
    listen 443 ssl;
    server_name api-aaa.abc.com;

    location / {
        include /etc/nginx/uwsgi_params;
        uwsgi_pass uwsgicluster  # 转发上游集群
    }

    ssl_certificate   /home/hsheng/www/hsheng-mall-backend/ssl_certs/api-aaa.abc.com_bundle.crt;
    ssl_certificate_key  /home/hsheng/www/hsheng-mall-backend/ssl_certs/api-aaa.abc.com.key;
    ssl_session_timeout 5m;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
    ssl_prefer_server_ciphers on;
}

server {
    listen 80;
    server_name api-aaa.abc.com;
    return 301 https://$host$request_uri;
}
```

可以看到，与单节点不同之处在于，新增了 `upstream` 定义，并将 `uwsgi_pass` 修改为定义的 `upstream` 名称。

在配置文件中，还能设置各个节点的权重分配等，此处不展开介绍，默认为轮询方式，请求随机派发到各节点。更多关于 Nginx 负载均衡的配置方式可见[『Nginx 负载均衡配置』](/2022/07/23/nginx-load-balance/)。