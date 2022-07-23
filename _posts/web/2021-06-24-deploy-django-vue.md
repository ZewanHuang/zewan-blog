---
layout:     post
title:      服务器部署 Vue 和 Django 项目的全记录
subtitle:   "A Record of the Whole Process of Deploying VUE and DJANGO Projects"
date:       2021-06-24 16:19:49
author:     "Zewan"
catalog:    true
header-style: text
target: _blank
tags:
    - Web
    - 部署
    - Nginx
    - Vue
    - Django
---

本篇记录我在一个全新服务器上部署 Vue 和 Django 前后端项目的全过程，内容包括服务器初始配置、安装 Django 虚拟环境、python web 服务器 uWSGI 和反向代理 Nginx 的使用，以及报错的纠正等。

若前后端采用的技术栈和我相同，可基本按照本文进行操作；否则可能需要理解所涉及步骤的意义和使用，再结合自己的技术栈进行调整。

## 服务器预设

### 租服务器

各大云平台，如腾讯云、阿里云、华为云等，都有学生优惠。我选择的是腾讯云，原因：UI好看。

我所租借服务器的配置如下，仅供参考：

![服务器配置](/img/in-post/post-deploy/server.png)

* 镜像信息：CentOS 7.6 64bit
* 实例规格：CPU 1核，内存 2GB
* 磁盘：系统盘 40GB
* 流量包套餐：带宽 5Mbps，流量包 1000GB/月（免费）

我使用的是 CentOS，关于 CentOS 和 Ubuntu 镜像的选择，可以参考 <a href="https://zhuanlan.zhihu.com/p/32274264" target="_blank">CentOS、Ubuntu、Debian三个linux比较异同 - 知乎</a>。

很多企业部署在生产环境的服务器使用的是 CentOS，但对于个人网站或者课内学习之用，我认为可能 Ubuntu 会方便一些也容易上手一些，从实操来看，很多 Ubuntu 能直接 apt 下载的东西，CentOS 要绕不少弯。

> 如果你选择的是 Ubuntu，这篇文章也是能给你部署带来帮助的，因为步骤大同小异

### SSH 远程连接

配置 SSH 远程连接，方便本地操作服务器，而无需每次都登录云平台。

在控制台中点击登录，进入服务器终端。第一步需要初始化超级用户 root 的密码，进入 superuser 权限。

``` bash
sudo passwd       # 初始化密码
su                # 切换到root超级用户
```

修改配置文件，允许密码或密钥远程连接。

``` bash
vim /etc/ssh/sshd_config      # 编辑ssh设置文件
```

在打开的文件中，修改：

``` bash
RSAAuthentication yes                       # 开启rsa验证，需要添加
PubkeyAuthentication yes                    # 开启公钥登录，一般被注释掉了，去掉前面的#就好
AuthorizedKeysFile .ssh/authorized_keys     # 公钥保存位置，原来就有
PasswordAuthentication yes                  # 开启使用密码登录
```

保存退出，重启 SSH 服务。

``` bash
service sshd restart        # 重启ssh服务
```

设置完毕后，即可在本地 powershell 或 git bash 连接服务器。

``` bash
ssh root@<IP address>       # IP address 为你服务器的公网IP地址
```

另外，VScode 的 [Remote - SSH](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh) 远程连接插件真香。

### 添加用户

所有命令都在 root 用户下执行，这样是不明智的，所以实现远程连接后，我们在本地终端连接服务器，使用以下命令添加一个新用户：

``` bash
adduser <username>
```

为其指定密码：

``` bash
passwd <username>
```

如果服务器是本人的，还可以为创建的用户添加 sudo 权限：

``` bash
vim /etc/sudoers
```

将 root 所在行复制后改为用户的 username，保存后该用户则拥有 sudo 权限；另外设置需要密码才能使用 sudo 权限，则设置后面字段为 `ALL`，不需要密码则为 `NOPASSWD:ALL`。修改后大致为：

``` bash
root      ALL=(ALL)       NOPASSWD:ALL
hadoop    ALL=(ALL:ALL)   ALL
```

### 配置公钥

配置公钥后，本地连接服务器，无需每次都输入密码。

首先，生成本地电脑的公钥。

``` bash
ssh-keygen -t rsa           # 打开cmd或powershell输入
```

默认回车即可，成功后在 `C:\Users\用户名\.ssh` 文件夹下会生成 `id_rsa` 和 `id_rsa.pub`，后者就是本地用户的密钥。打开该文件，复制内容。然后使用 ssh 命令登录远程服务器，在用户根目录下（`~/`）创建 .ssh 文件夹并进入，再创建 authorized_keys 文件，将密钥粘贴进去，之后重启 ssh 服务。

``` bash
service sshd restart        # 重启ssh
```

### 更新系统软件包

服务器的预配置都比较古老，依次输入以下命令升级软件包或依赖。

``` bash
yum update -y                               # 更新系统软件包
yum -y groupinstall "Development tools"     # 安装软件管理包
yum install openssl-devel bzip2-devel expat-devel gdbm-devel readline-devel sqlite-devel psmisc libffi-devel epel-release     # 安装可能使用的依赖
```

## 配置 Django

本步骤中，我在服务器上搭建 Django 环境，采用的是 virtualenv 虚拟环境管理器。当然我现在重新配置的话，可能不再会采用该方式了，我更推荐安装 <a href="https://www.anaconda.com/" target="_blank">Anaconda</a> 或者 <a href="https://docs.conda.io/en/latest/miniconda.html" target="_blank">Miniconda</a>（服务器比较小型可选择后者），通过 Conda 来管理 Python 环境会方便一些。

> TODO: 也许我什么时候会想写篇博客简单介绍一下 Conda

### 安装 python3.8.4

CentOS 拿到手，发现不自带高版本 Python 时，我是很懵的，这也是我推荐入门者用 Ubuntu 的原因之一。推荐归推荐，当初我还是乖乖地给自己安了个 Python 3.8。

在执行以下操作前，请先输入 `python -V` 查看一下本地 Python 版本，如果是 3.x 这一步就不需要做了。

``` bash
cd /usr/local                   # 我一般喜欢把文件下载到该目录下
wget https://www.python.org/ftp/python/3.8.4/Python-3.8.4.tgz
tar -zxvf Python-3.8.4.tgz      # 解压python包
```

进入 Python 包路径，并编译安装到指定路径 /usr/local/python3

``` bash
cd Python-3.8.4
./configure --prefix=/usr/local/python3
make && make install
```

安装成功后，建立软链接，添加环境变量。因为服务器系统自带有 python、python2、python3，因此我命名为 python3.8，避免冲突。但我的服务器只有 pip3 没有 pip，所以我将 pip3.8 的软连接命名为 pip。

``` bash
ln -s /usr/local/python3/bin/python3.8 /usr/bin/python3.8
ln -s /usr/local/python3/bin/pip3.8 /usr/bin/pip
```

检测是否安装成功。

``` bash
python3.8 -V
pip -V
```

### 安装虚拟环境

建议安装虚拟环境 virtualenv，当不同项目要求的 python 版本不同时，不会产生冲突。

``` bash
pip install virtualenv
pip install virtualenvwrapper       # 管理虚拟环境
```

下载成功后，创建存储虚拟环境的目录。

``` bash
mkdir ~/.virtualenvs                # 我一般存放在 /root/.virtualenvs，可自行修改
```

查找 `virtualenvwrapper.sh` 文件位置，添加环境。

``` bash
find / -name virtualenvwrapper.sh
```

编辑 `.bash_profile` 文件，在末尾添加这两句，其中 `source` 后的路径为前面查到的路径。

``` bash
export WORKON_HOME=$HOME/.virtualenvs
source  /usr/local/python3/bin/virtualenvwrapper.sh
```

保存修改后，更新配置信息。

``` bash
source ~/.bash_profile 
```

如果保存时报错，在 /etc/profile 中加入下面内容，再 `source`。

``` bash
export VIRTUALENVWRAPPER_PYTHON=/usr/bin/python3.8
export VIRTUALENVWRAPPER_VIRTUALENV=/usr/local/python3/bin/virtualenv
```

### 创建虚拟环境

通过 -p 指定使用的Python版本，创建成功后自动进入该虚拟环境。

``` bash
mkvirtualenv -p python3.8 django        # django为虚拟环境名称
```

如果你希望将当前虚拟环境安装的所有插件配置到新虚拟环境中，可以执行：

``` bash
pip freeze > requirements.txt           # 导出依赖
pip install -r requirements.txt         # 进入新虚拟环境后再执行
```

**虚拟环境的其它常用命令**

- 查看创建的全部虚拟环境：`workon`
- 使用某一虚拟环境：`workon 虚拟环境名称`
- 退出当前虚拟环境：`deactivate`
- 删除虚拟环境：`rmvirtualenv 虚拟环境名称` 记得退出再删除

### 虚拟环境中安装 Django 和 uWSGI

uWSGI 可以理解为服务器上持续运行 Django 的代理服务器，用于与 Django 后端进行数据传输等，后续配置需要使用。

进入前面创建的虚拟环境，安装。

``` bash
pip install django==3.2         # 可指定版本
pip install uwsgi
```

> uWSGI 要安装两次，一次在虚拟环境中，另一次退出虚拟环境进行安装

创建 uWSGI 的软链接。

``` bash
ln -s /usr/local/python3/bin/uwsgi /usr/bin/uwsgi
```

## 安装 Nginx

Nginx 是 Http 反向代理 web 服务器，同时也提供 IMAP/POP3/SMTP 服务，占用内存少，并发能力强。在这里我们只需要了解，Nginx 能帮我们在指定端口跑我们的项目就好了。

``` bash
yum install nginx
```

安装成功后，相关的文件存储路径为

- 安装成功后，默认的网站目录为 `/usr/share/nginx/html`
- 默认的配置文件为 `/etc/nginx/nginx.conf`
- 自定义配置文件目录为 `/etc/nginx/conf.d/`

在启动之前，还需确保服务器的相关端口已打开。http 对应 80 端口，https 对应 443 端口。一般在云平台租的服务器，可以在控制台中的防火墙处开启相应端口。我的设置可供参考。

![服务器端口](/img/in-post/post-deploy/duankou.png)

接下来启动 Nginx

``` bash
systemctl start nginx
```

启动成功后，浏览器搜索服务器 IP 地址，就能访问到 Nginx 主页了。

![Nginx 默认主页](/img/in-post/post-deploy/Nginx.jpg)

## 部署项目

### 上传项目

Django 后端项目文件，直接上传至服务器即可。Vue 框架写的前端，需要使用 `npm run build` 命令进行打包，再将生成的 dist 目录上传。

这里推荐软件 [FileZilla](https://filezilla-project.org/)，用于本地与服务器文件传输十分方便。

### 配置 uWSGI

新建文件 uwsgi.ini，我习惯放置于 Django 项目的根目录下，用于指定项目路径、最大进程数、运行端口等。我的配置参数可供参考。

``` ini
[uwsgi]
socket = 127.0.0.1:8080
chdir = /root/Ops/django
wsgi-file = /root/Ops/django/django3/wsgi.py
master = true 
enable-threads = true
processes = 8
buffer-size = 65536
vacuum = true
daemonize = /root/Ops/django/uwsgi.log
virtualenv = /root/.virtualenvs/django
uwsgi_read_timeout = 600
threads = 4
chmod-socket = 664
```

简要介绍该文件的配置信息：

- `[uwsgi]`：必须有这个[uwsgi]，不然会报错
- `socket`：该端口为后端 Django 的运行端口，可自定义，但须与后面 Nginx 的配置一致
- `chdir`：django 项目路径
- `wsgi-file`：django 项目的 wsgi.py 文件路径
- `master`：开启主进程
- `processes`：最大进程数量
- `vacuum`：当服务器退出的时候自动删除 unix socket 文件和 pid 文件
- `daemonize`：输出日志，有报错时可查看
- `virtualenv`：项目虚拟环境路径

切换当前路径到 uwsgi.ini 文件所在目录，启动 uWSGI。

``` bash
uwsgi --ini uwsgi.ini
```

使用 `ps` 命令查看进程，检测是否成功。

``` bash
ps -aux | grep uwsgi
```

![uwsgi 进程查看](/img/in-post/post-deploy/uwsgi.png)

### 配置 Nginx

> 这里我部署的是域名而非IP，IP配置与域名的区别在于，不需要SSL证书字段。

首先，删除 `/etc/nginx/nginx.conf` 文件中 `server{...}` 部分的代码。当然，如果怕出错，也可先将原本的 nginx.conf 文件备份一下。

接下来，在 `/etc/nginx/conf.d` 文件夹中修改默认文件 `default.conf`（若不存在则新建一个），文件内容如下：

``` conf
server {
    listen 80;
    listen 443 ssl;
    server_name  zewan.top www.zewan.top;

    location / {
        root /root/Ops/vue/dist;
        index index.html index.htm;
        try_files $uri $uri/ /index.html;
    }

    location /api {        
        include /etc/nginx/uwsgi_params;
        uwsgi_pass 127.0.0.1:8080;                                                               
    }

    ssl_certificate /etc/nginx/ssl/zewan.top.crt;
    ssl_certificate_key /etc/nginx/ssl/zewan.top.key;
    ssl_session_timeout  5m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4:!DH:!DHE;
    ssl_prefer_server_ciphers  on;

    error_page 497  https://$host$uri?$args;
}
```

简要说明文件内容的作用：

- `listen` 后接端口，即设定访问的端口，此处同时开放 80 和 443
- `server_name` 为访问域名
- `location /` 后描述前端 dist 项目文件夹的存放地址，**需根据自身情况修改**，注意 dist 即为前端项目的根目录
- `location /api` 后为后端项目运行端口，注意 `uwsgi_pass` 后须与之前 uWSGI 的配置保持一致
- `ssl_certificate[_key]` 为 SSL 证书存储路径

**重要提醒**

采用 `location /api` 与 uWSGI 连接，最终将后端运行在 `:443/api/`。需保证后端的路由都是 `api/*`，即 Django 项目的 `urls.py` 文件所有路由前需加 `api/`。

## 运行项目

检测 Nginx 配置是否有误，成功后重启 Nginx 服务。

``` bash
nginx -t                # 测试
nginx -s reload         # 重新加载
```

**注意**，若修改了后端 Django 内容或其它内容，须重启 uWSGI 和 Nginx 服务，否则不生效！

``` bash
ps -ef | grep uwsgi         # 查看uWSGI进程
killall -9 uwsgi            # 用kill方法把uwsgi进程杀死
uwsgi --ini uwsgi.ini       # 重启uwsgi
nginx -s reload             # nginx平滑重启
```

另外，如果你的项目文件存放于 root 用户目录下，访问网站时可能出现 500 或 403 Forbidden 权限报错，此时需修改 `/etc/nginx/nginx.conf`，将文件首行的 `user nginx` 修改为 `user root`。

至此网站已部署完毕，欢迎访问[我的网站](https://qn.zewan.cc)。项目[网上出版系统](https://github.com/BUAASE-Slime/Questionnaire-Planet)已开源，欢迎交流学习！