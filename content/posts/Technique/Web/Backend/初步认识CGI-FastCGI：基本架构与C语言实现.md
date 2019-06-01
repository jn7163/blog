---
title: 初步认识CGI/FastCGI：基本架构与C语言实现
date: 2018-12-30 12:33:47
tags:
  - c
  - cgi
---

最近在做 C 语言课大作业，一拍脑门就决定用 C 语言来实现一个 CGI 程序 —— 这是真实“文案一时爽，开发火葬场” 。总之花了一天时间总算是写出来了，虽然其中大部分时间都是在折腾 sqlite3 那些难懂的接口 。这篇博客就简单记录一下以下几个收获：

- CGI 与 FastCGI 的概念与区别

- 处理简单的 GET/POST 请求

源代码同样贴在 [GitHub](https://github.com/queensferryme/ccgi)，欢迎参观 。

<!--more-->

## 认识 CGI / FastCGI

### CGI 的出现

故事要从早些时候说起 。

在 HTTP / HTML 刚被提出的时候，Web 服务器通常只承担着分发静态网页的责任 。但随着时间的流逝，用户对万维网的需求也在不断增加 —— 这其中就包括动态渲染网页的需求 。例如，用户在某个购物网站上对商品进行评价，而这些评价被存储在数据库的内部；当用户想要在网页上看到该商品的最新评价时，我们就必须有某个程序从数据库中选取相应数据并将它们动态地渲染成网页才行 。这就是 CGI 的使命 。

CGI 程序是独立于 Web Server 的中间件，它能起到连接服务器与数据库的作用 。下面这张图简单的说明了 CGI 的工作流程：

![User-Server-CGI](https://i.loli.net/2018/12/30/5c2854c26d339.png)

更准确的说，CGI 是一种 Web 服务器与应用程序之间的协议 —— 这个协议定义了二者之间数据传输的方式 。例如，使用**环境变量**来交换请求参数，使用**STDIN、STDOUT、STDERR**进行输入输出 。下面就给出一个简单的C 语言 CGI 程序：

```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define CHUNKSIZE 1024

int main() {
    char post_data[CHUNKSIZE];
    fgets(post_data, strtol(getenv("CONTENT_LENGTH"), NULL, 10) + 1, stdin);
    printf("Content-Type: text/html\r\n"
           "\r\n"
           "<b>POST DATA</b>: %s"
           "<b>QUERY STRING</b>: %s",
           post_data, getenv("QUERY_STRING"));
    return 0;
}
```

虽然这程序看起来丑得难以置信（<sub>小声</sub>），但它的确是一个 CGI 程序 。它通过 STDIN 读取 POST 方法发送的数据，并将生成的 HTTP 响应通过 STDOUT 返回给 Web 服务器，其中还有一些参数（比如 `CONTENT_LENGTH` 与 `QUERY_STRING`）通过环境变量传递给 CGI 程序 。如果你使用 lighttpd 或是 Apache 之类的服务器进行相应的配置，你就可以访问这个网页了 。

### 更快的 FastCGI

CGI 的出现是个极大的进步，但是它也很快遇到了新的瓶颈 —— 性能问题 。CGI 协议采用了 `fork-and-execute` 的执行模式，对于没一个请求都会启动一个新的进程；而这个进程一旦处理完相应的请求就会自动退出 。这种执行方式低效且占用系统资源（<sub>想想双十一晚上你需要启动多少进程</sub>），于是就有了 FastCGI 。FastCGI 在原有的基础上增加了一个统一的进程管理调度机制 —— 它管理着多个子级 CGI 解释器，并且能做到高效的请求分配和进程复用，从而是服务器性能有了很大的提升 。

古老的故事就到此为止了，下面讲讲具体的实现 。

## 实现 FCGI ！

### 安装依赖

- `spawn-fcgi`
  - 作用：用于生成 FCGI 进程
  - 安装
    - Ubuntu：`apt-get install spawn-fcgi`
    - Others：https://github.com/lighttpd/spawn-fcgi
- `fcgi`
  - 作用：实现了 FCGI 协议的开发标准库
  - 安装
    - Ubuntu：`apt-get install libfcgi-dev`
    - Others：https://fastcgi-archives.github.io/
- `sqlite3`
  - 作用：sqlite3 数据库开发件
  - 安装
    - Ubuntu：`apt-get install libsqlite3-dev`
    - Others：https://www.sqlite.org/download.html

### 路由与视图函数绑定

参考 Flask 与 Django 的一些开发经验，我设计出这样相对清晰的业务处理逻辑：

```c
int main() {
    while(FCGI_Accept() >= 0) {
        char *route = getenv("DOCUMENT_URI")，
             *query_string = getenv("QUERY_STRING"),
             *method = getenv("REQUEST_METHOD");
        if(equal(route, "/user") && equal(method, "GET"))
            user(query_string);
        else if(equal(route, "/admin") && equal(method, "GET"))
            admin();
        else if(equal(route, "/update") && equal(method, "POST"))
            update(query_string);
        else if(equal(route, "/delete") && equal(method, "GET"))
            delete(query_string);
        else sysinfo();
    }
    return 0;
}
```

需要注意的是在这种情况下，`/user` 与 `/user/` 是不同的 —— 在上面这个程序中我只匹配了 `/user` 路径 。

### 从 STDIN 处理 POST 数据

从标准输入流读取数据并不困难，但在 FCGI 环境下有一个小坑 —— Web 服务器提供的输入流不保证能够有正确的终止符号，例如 EOF 或者空字符 `\0` 等 。所以我们需要根据 `CONTENT_LENGTH` 环境变量来读取对应数量的数据 。

```c
int content_length = strtol(getenv("CONTENT_LENGTH"), NULL, 10);
fgets(post_data, content_length + 1, stdin);
```

### 编译与部署

项目并不复杂，只有 `main.c` 、`routers.h`  、`utils.h` 三个文件，只需要写好 Makefile 编译即可 。编译时参数有 `-l sqlite3` 和 `-lfcgi` 以链接相应库 。假设编译生成的 CGI 程序为 `main.cgi` 。

编译成功后，就可以使用 `spawn-fcgi` 命令来启动 FCGI 进程 。例如：`spawn-fcgi -a 127.0.0.1 -p 5000 -f $PWD/main.cgi -P main.pid` 。这个命令会从 `main.cgi` 启动 FCGI 进程并监听本地的 5000 端口，进程的 pid 同时也会被保存在当前目录的 `main.pid` 文件内 。

<div class="note danger">**注意**：`spawn-fcgi` 中可执行文件应该使用绝对路径 。</div>

最后的部署我采用了个人最喜欢的高性能 Web 服务器 Nginx 。服务器配置只有简单的几行：

```nginx
server {
    listen 4000;
    server_name localhost;
    location / {
        fastcgi_pass 127.0.0.1:5000;
        fastcgi_index index.cgi;
        include /etc/nginx/fastcgi.conf;
    }
}
```

（*不说了，复习线性代数去了qwq*）

______

**参考文献：**

- [博客园：CGI与FastCGI](https://www.cnblogs.com/wanghetao/p/3934350.html)
- [博客园：Nginx + CGI/FastCGI + C/Cpp](https://www.cnblogs.com/skynet/p/4173450.html)
- [FastCGI-Archives：FastCGI Developers‘ Kit](https://fastcgi-archives.github.io/FastCGI_Developers_Kit_FastCGI.html)