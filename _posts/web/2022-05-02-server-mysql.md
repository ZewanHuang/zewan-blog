---
layout:     post
title:      Ubuntu 服务器安装 MySQL 远程数据库
subtitle:   "Install MySQL Remote Database on Ubuntu Server"
date:       2022-05-02 15:47:39
author:     "Zewan"
catalog:    true
mathjax:    true
header-style: text
tags:
    - MySQL
    - Linux
    - Web
---

在 Web 项目中，我们需要使用到远程数据库，开发阶段也需要连接并查看数据库的状况。腾讯云、阿里云等云平台提供了远程数据库，可直接使用；当然也可以自己在部署 Web 的服务器上安装数据库，将其配置为远程数据库，供 Web 应用使用。

本篇介绍如何在 Linux 服务器上安装 MySQL 数据库，并设置为可远程连接。

## 在 Ubuntu 上安装 MySQL

为安装最新版本的 MySQL，我们可以先更新一下 apt 管理的资源包。

以 sudo 用户身份登录，执行以下命令：

```bash
sudo apt update
```

待更新完毕后，输入以下命令，安装 MySQL：

```bash
sudo apt install mysql-server
```

安装完成后，MySQL 服务会自动启动。想验证 MySQL 正在运行，输入：

```bash
sudo systemctl status mysql
```

输出如下图，即表示已启动。

![MySQL已启动](/img/in-post/post-ubuntu-mysql/mysql-running.png)

## 开启远程连接权限

### 编辑 MySQL 配置文件

默认情况下，MySQL 数据库仅监听本地连接。若想允许远程连接数据库，首先需要修改配置文件，让 MySQL 可以监听远程固定 IP 或所有远程 IP。

配置文件 `mysqld.cnf` 路径一般为 `/etc/mysql/mysql.conf.d/mysqld.cnf`。

输入以下命令打开编辑：

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

找到 `bind-address` 一行，默认该值为 127.0.0.1，仅监听本地连接。我们将其改为远程连接 IP 可访问，可以使用通配符 IP 地址 `0.0.0.0`，也可以是固定 IP，仅允许指定 IP 连接。这里我修改为 `0.0.0.0`，允许所有 IP 地址访问。

![mysqld.cnf](/img/in-post/post-ubuntu-mysql/mysqld_cnf.png)

> 在某些 MySQL 版本的配置文件中，没有 `bind-address` 一行，在如上图的合适位置上添加即可。

更改后，保存并退出编辑器（使用 Ctrl+X 保存并退出）。后重启 MySQL 服务，使新配置生效。

```bash
sudo systemctl restart mysql
```

### 创建 MySQL 用户

以 sudo 权限进入 MySQL 服务：

```bash
sudo mysql
```

进入 MySQL 后，创建一个可远程连接 MySQL 的用户，并设置为使用密码作为认证方式。

```sql
CREATE USER 'zewan'@'%' IDENTIFIED WITH mysql_native_password BY 'zewan1234';
```

上述命令中，`%` 表示 IP 任意，`@` 前的用户名和 `BY` 后面的密码修改为自己的信息。

执行完毕后，使用下列命令可以查看到所有的 user，包括我们新建的：

```sql
SELECT DISTINCT CONCAT('User: ''', user, '''@''', host, ''';') AS quert FROM mysql.user;
```

![mysql_users](/img/in-post/post-ubuntu-mysql/mysql-users.png)

接下来，我们赋予该用户拥有所有数据库的访问权限，使其成为新的独立管理用户：

```sql
GRANT ALL PRIVILEGES ON *.* TO 'zewan'@'%' WITH GRANT OPTION;
```

最后，刷新 MySQL 系统权限相关表，更新缓存，并退出 MySQL。

```sql
FLUSH PRIVILEGES;
EXIT;
```

## 远程连接 MySQL 数据库

### 命令行远程访问

命令格式如下：

```bash
mysql -u <username> -h <mysql_server_ip> -p
```

### Jetbrains 家族 Database 连接

在 IDEA、Pycharm 等软件中，内置 Database 访问插件，具备可视化数据库表的功能，一般在右侧任务栏点击展开。

点击加号，选择 MySQL 作为 Data Source。

![pycharm-database](/img/in-post/post-ubuntu-mysql/pycharm-database.png)

在弹出框中，填入远程数据库IP(Host)、用户名(User)、密码(Password)，后点击 Test connection 尝试连接。出现下图成功标识，即表示可成功连接数据库，随后点击应用(Apply)即可。

![pycharm-success](/img/in-post/post-ubuntu-mysql/pycharm-success.png)

随后，软件中会出现 console，我们可以在这里输入 MySQL 语句并点击绿色启动按钮执行命令，同时可双击右侧弹出栏中的数据库表，查看信息。

---

附上 MySQL 创建数据库，并指定编码 UTF8 的命令：

```bash
CREATE DATABASE `mydb` CHARACTER SET utf8 COLLATE utf8_general_ci;
```
