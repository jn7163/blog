---
title: Let's Encrypt 证书配置：使用 acme.sh 自动化工具
date: 2018-11-28 18:22:45
tags:
  - https
  - nginx
---

最近学习浏览器的 Notification API，想要完成一个具有类原生推送功能的页面 。但在部署过程中发现 Chrome 等现代浏览器大多只允许 https 网站发送通知，所以借此机会配置一下 Nginx 服务器的 ssl 访问 。我选择的是开源免费的 Let's Encrypt 证书和一套非常方便的 acme (Automatic Certificate Management Environment) 工具 [acme.sh](https://github.com/Neilpang/acme.sh) 。

<!--more-->

## Let's Encrypt 验证原理

首先了解一下 Let's Encrypt 的验证原理 。

传统的 ssl 证书都是通过向网站管理员邮箱发送验证邮件来实现的；但 Let's Encrypt 是在你的服务器上生成一个随机文件，然后通过访问你的 HTTP 站点的 `/.well-known/acme-challenge/<hash-tag>` 路径来验证你是否具有对该域名的控制权 。例如，如果你把你的验证文件保存在 `/var/www/challenge` 目录下，你可能需要这样填写服务器配置（以 Nginx 为例）：

```nginx
server {
    server_name example.com;
    location ^~ /.well-known/acme-challenge/ {
        alias /var/www/challenge;
        try_files $uri =404;
    }
    location / {
        rewrite ^/(.*)$ http://example.com/$1 permanent;
    }
}
```

这样在访问你的站点的 `/.well-known/acme-challenge/<hash-tag>` 路径会优先从 `/var/www/challenge` 下查找该随机生成文件：如果文件获取成功，则证明你的身份；如果文件不存在，那么你就无法获得 ssl 证书 。

## 如何申请证书

由于安全方面的考量和证书的开源免费性质，Let's Encrypt 证书通常只有 90 天的有效期 。Let's Encrypt 官方提供了 [certbot](https://certbot.eff.org/) 工具来自动化地申请和更新证书，但这里我推荐使用 [acme.sh](https://github.com/Neilpang/acme.sh)：它几乎提供了一站式的证书自动化解决方案 。执行命令`curl https://get.acme.sh | sh` 即可安装工具 。

### 普通的申请

下面以一个最简单的静态文件分发服务器为例 。假设你的站点根目录在 `/var/www/example.com`，那么你可以通过以下命令来来申请证书：`acme.sh --issue -d example.com -d www.example.com -w /var/www/example.com`；这个证书文件将对 `example.com` 与 `www.example.com` 有效。如果一切无误，你就会通过验证并得到证书文件；期间你完全无需手动修改配置，acme.sh 会帮你自动打理好一切 —— 修改配置、设定好 crontab 自动更新、再将配置恢复原样 。

申请完证书后，你依然需要安装证书 。你可以使用以下命令来安装刚刚获取的证书：

```
acme.sh --installcert -d example.com \
--key-file /etc/nginx/ssl/example.com.key \
--fullchain-file /etc/nginx/ssl/example.com.cer \
--reloadcmd "service nginx force-reload"
```

然后你就可以在 `/etc/nginx/ssl` 目录下看到证书文件和密钥文件了 。

<div class="note primary">注意：你必须要有对站点根目录的**写权限**，通常来说这意味着你需要 `root` 权限；不要直接使用 `~/.acme.sh/example.com/` 目录下的证书文件，而应使用 `--install-cert` 命令 。</div>

### 使用 DNS API

许多 DNS 服务提供商都有提供 API，acme.sh 也能使用这些 API 来实现更加自动化的证书申请和更新 。例如我的腾讯云服务器使用的是 DNSPod 提供的解析服务，所以我只需要登录 DNSPod 并在用户中心的安全设置里获取自己的 API Token，然后执行以下命令来获取证书：

```bash
export API_Id="<your-id>"
export API_Key="<your-key>"
acme.sh --issue --dns dns_dp -d queensferry.cn -d '*.queensferry.cn'
```

这里我使用了泛域名解析，这样所有二级域名都可以使用这一份证书 。对于其他的 DNS 服务商也基本都有相应的 API ，可以在[这里](https://github.com/Neilpang/acme.sh/tree/master/dnsapi)查看 acme.sh 支持的 DNS API 列表 。**注意 DNSPod 只会显示一次你的 API Token**，所以务必要复制或截图保存 。

## 修改 Nginx 配置文件

### 配置 ssl 连接

安装完证书后，你就可以配置 Nginx 来升级你的站点了 。这里就直接贴出我的简单的配置：

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl on;
    ssl_certificate /etc/nginx/ssl/example.com.cer;
    ssl_certificate_key /etc/nginx/ssl/example.com.key;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers HIGH:!aNULL:!MD5;

    root /var/www/example.com;
    index /index.nginx-debian.html;
}
```

最重要的部分就是配置好 `ssl_certificate` 和 `ssl_certificate_key` 的路径 。

### http 自动跳转 https

配置完后你可能会发现，我们只有在访问 443 端口时才会使用 https 协议 。想要强制使用 https 来访问你的站点，一种方法是直接将域名挂到 HSTS 上；另一种方法就是将所有 http 访问重定向到 https 站点上 。例如我配置了这样一个跳转服务器：

```nginx
server {
    listen 80;
    server_name example.com;
    rewrite ^(.*)$ https://$host$1 permanent;
}
```

这样所有 http 访问都会被重定向到 https 站点上 。

------

**参考文献**

- [Let's Encrypt，免费好用的 HTTPS 证书](https://imququ.com/post/letsencrypt-certificate.html)

- [腾讯云 DNSPod API 申请 Let’s Encrypt 泛域名证书](https://cloud.tencent.com/developer/article/1064471)

- [nginx 配置 http 强制跳转 https](https://docs.lvrui.io/2017/04/01/nginx%E9%85%8D%E7%BD%AEhttp%E5%BC%BA%E5%88%B6%E8%B7%B3%E8%BD%AChttps/)

