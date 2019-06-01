---
title: Docker 初探：部署 MySQL 与 Flask 服务
date: 2018-10-30 08:38:23
tags:
  - docker
  - flask
  - mysql
---

Docker 是目前很流行的应用容器化技术；它能以小得多的资源占用实现系统级别的虚拟化，达到类似虚拟机的效果。使用 Docker 部署自己的应用十分方便而强大：你既不需要费心于各种与应用无关的繁琐配置；又同时拥有对应用高度的自由化管理权限。这篇博客记录一下我初步学习使用 Docker 的成果。

<!--more-->

## 安装 Docker

在 Arch Linux 上，安装 Docker 十分简单。你只需要运行`sudo pacman -S docker`即可获得一个完整的 Docker 应用。关于其他系统的安装信息可以参见 [Docker 教程](http://www.runoob.com/docker/docker-tutorial.html) 。

初步安装完 Docker 后一个比较常见的问题就是 Docker 运行权限 。许多用户会发现每次运行任何一条 Docker 指令都需要`sudo`提取超级用户权限实在是太过低效恼人。对此你可以创建一个特殊的`docker`用户组，这个用户组里的成员将拥有无需`sudo`直接运行的权限。具体操作如下：

```bash
sudo groupadd docker
sudo usermod -aG docker $USER
```

完成这两条指令后你就能自由地以非超级用户身份运行 Docker 啦 。如果你需要更详细的教程，请参考[官方文档](https://docs.docker.com/install/linux/linux-postinstall/) 。

## 基本概念

Docker 中的两个最主要的概念就是镜像（Image）与容器（Container）；这两者之间的关系有些类似于面向对象编程语言里类与对象的关系。你可以通过镜像来创建一个容器实例，而这个容器内部就搭载着你所需要的 app 。由于 DockerHub 上大量热心的贡献者，我们在部署许多应用时都有可参照甚至开箱即用的镜像。下面就来尝试运行一个最简单的 Hello World 应用：

```bash
docker pull hello-world
docker run hello-world
```

如果运行成功，你就能看到来自 Docker 的问候 。你可以通过`docker images`命令来查看你本地所有镜像，通过`docker ps -a`来查看本地所有（无论是否仍在运行）容器。

## 运行容器

接下来我尝试使用 Docker 容器化技术部署一个相对完整可用的 MySQL 数据库。我们当然不需要从头构建一个 MySQL 镜像，只需使用 DockerHub 上的官方镜像即可。具体操作如下：

```bash
docker pull mysql:latest
docker run -p 3000:3306 --name mysql-dev -e MYSQL_ROOT_PASSWORD=123456 -d mysql:latest
```

一个完整可用的数据库已经搭建起来了！解释一下这两条指令其中的具体流程：首先我们从 DockerHub 上拉取了一份 MySQL 镜像；这份镜像的标签是`latest`，即表明这是最新版本的 MySQL 镜像（你当然也可以使用其他版本标签）。随后我们就通过`docker run <image:tag>`指令来创建一个容器，我们使用：

- -p 绑定端口。由于 Docker 容器与宿主机的环境是相对隔离的，我们必须将相应的端口暴露出来才能连接上 MySQL 服务器 。这里我们将容器内部的 3306 端口（MySQL 的默认端口）与宿主机的 3000 端口绑定在一起，这样你就能通过访问宿主机的 3000 端口操作数据库 。
- --name 给容器命名。这里我们将容器命名为 mysql-dev；在生产环境中我显然会采用更严格的数据库密码以及其他措施来保证安全。
- -e 指定环境变量。这里我们设置了 MySQL 的密码。
- -d 使容器在后台（daemon）运行。

这样一个完整的 MySQL 服务器就在容器内运行起来了。

## 维护容器

### 端口映射

上一个容器的一个主要安全问题就是我们将容器内部端口映射到了宿主机的公网 IP 上。我们可以通过以下命令来查看容器的端口映射：

```bash
docker port mysql-dev
# 3306/tcp -> 0.0.0.0:3000
```

对于生产数据库，我们可能并不想在公网上开放它的端口 。在这种情况下，我们可以用`docker run -p 127.0.0.1:3000:3306 -d mysql:latest`来将端口映射到宿主机的 localhost 上。

### 容器网络

在许多应用场景下，我们将应用与数据库分别部署在两个容器内。这时候想要使得应用能够正常地访问数据库，最优雅的解决方案就是使用 network 。我们可以创建一个网络将容器联系起来，容器之间可以互相通过容器名来代替 IP 访问。这里的容器名类似与 localhost，是一个记录在 host 文件里的特殊的地址。

假设现在我有两个容器：生产数据库`mysql`与应用容器`app`。我们可以通过以下命令来创建网络并连接容器：

```bash
# create a docker network
docker network create app
# connect to the docker network
docker network connect app mysql
docker network connect app app
```

这是`app`容器内就可以通过`<user>:<passwd>@mysql:<port>/<database>`来访问数据库。**注意：这里的`<port>`是指容器内部的端口，而非映射到宿主机上的端口。**

你也可以在启动容器时加入`--network=<name>`参数来将容器直接连接到网络内。

### 目录挂载

如果你有一个应用包含上传功能，而这个应用部署在 docker 容器内部；那么正常情况下用户所上传的文件会被存储在 docker 容器内部的文件系统里。你当然可以用`docker cp <container>:<volumn> <volumn>`命令来讲容器内部的文件拷贝到宿主机上 —— 但更优雅的做法是使用`-v`参数来挂载目录。例如你现在有一个应用的镜像，你可以这样在启动容器时挂载目录：

```bash
docker run -v /uploads:/uploads -d app
```

这样宿主机上`/uploads`目录里的任何变更都会反映在容器内的`/uploads`目录上，反之亦然。

## 创建镜像

对于一个持续开发的 app，我们的容器也需要持续更新 。所以每次 app 更新时，我们都需要重新构建镜像并启动容器。创建镜像的方式大体有两种：手动修改（直接模改原有镜像并提交）和使用 Dockerfile 自动构建。下面逐一介绍。

### 手动修改

假设我们手里有一个 python:3.5 镜像，而现在我们需要一个加载了一些第三方依赖库的 mypy 镜像。我们可以使用`docker run -it python:3.5 /bin/bash` 命令来交互式地操作容器内部系统；这里的 `-i` 参数表明容器从宿主机的标准输入接受输入，`-t` 参数表明容器 会打开一个伪终端，`/bin/bash` 是我们要运行的命令（亦即 entrypoint ）。假设我们在容器内进行了如下操作：

```bash
pip install Flask Flask-SQLAlchemy Flask-RESTful
exit
```

安装完以上依赖库，我们就可以提交容器使之成为新的镜像：`docker commit -m="mypy" -a="queensferry" d45f3e6fa9e8 mypy:latest` 。其中`d45f3e6fa9e8`是你的容器 ID，你可以通过`docker ps -a`来查看之前进行了更改的容器的 ID 。

这时再运行`docker images` 命令，你就能看到你的 `mypy:latest` 镜像。

### 自动构建

但是每次更新都重头构建镜像实在太过于繁琐 。一般而言，我们更多使用 Dockerfile 进行自动构建。下面贴出我为自己的 [Flask 应用](https://github.com/queensferryme/cobyr)编写的简单的 Dockerfile 作为入门参考，需要完整教程的可以参考[官方文档](https://docs.docker.com/engine/reference/builder/) 。

```dockerfile
FROM python:3.5
WORKDIR /

COPY cobyr cobyr
COPY requirements.txt requirements.txt
COPY boot.sh boot.sh
# COPY ~/.pip/ /root/.pip
RUN pip install --upgrade pip
RUN pip install -r requirements.txt
RUN pip install gunicorn
RUN chmod +x boot.sh

EXPOSE 5000
ENTRYPOINT [ "./boot.sh" ]
```

简单解释这些指令：

- FROM 指令指定了源镜像 。我们大多情况下仍然要通过模改现存的镜像来创建新镜像。

- WORKDIR 指令指定了镜像内部的默认工作目录。以下所有指令均已这个目录为上下文。

- COPY 指令将宿主机上的文件（夹）拷贝到镜像内部对应文件（夹）。

- RUN 指令能运行一条命令。

- EXPOSE 指令指定了镜像内部允许暴露映射到宿主机上的端口。

- ENTRYPOINT 指定了 `docker run <image:tag>` 时的默认命令，你当然也可以用 `/bin/bash` 等其他命令代替入口点。

在你的应用目录下创建 Dockerfile 并执行 `docker build -t app:latest .` ，docker 就会根据当前目录下的 Dockerfile 构造出名为 `app:latest` 的全新镜像。

## 小结

突然想扯一些与技术无关的事。说回来这是我上大学以来的第一篇博客，大学生活比想象中的忙碌很多。虽然这是第一次自己完全对自己负责，但上进心和自主性反倒前所未有的被激发了 —— 不需要再应付老师或家长或不感兴趣的琐事，只是全身心地为自己的梦想投入。细细想来，中国的高中教育所缺乏的是否就是这种为自己负责的学习机制呢？

以上。