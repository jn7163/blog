---
title: Termux 安装配置：Android 上的 Linux 终端
date: 2018-07-09 21:29:11
tags:
  - android
  - terminal
---

Termux 是一款强大的 Android 终端模拟器。它会在你的手机上安装一个无需 root 权限即可使用的最小化 Linux 系统，并且支持 apt 等包管理器 。常见的用法就是通过 SSH 将手机与 PC 端连接起来 。今天就记录一下 Termux 的安装配置流程。

<!--more-->

## 通过 SSH 连接

Termux 可以在 Google Play 上安装，安装完以后就能进入终端界面了。由于通过 PC 操作会更方便一些，这里先进行 SSH 的安装配置。

1. 更新 apt 并安装 openssh

   初次运行 Termux，用户需要先更新 apt 源列表。执行以下命令：

   ```bash
   apt-get update
   apt-get upgrade
   apt-get install openssh
   ```

2. 生成公钥并导入 Termux

   **出于安全考量，Termux 禁止使用用户名+密码访问 SSH 服务，必须使用密钥；且默认 SSH 端口为 8022。**

   在 PC 端执行`ssh-keygen` 生成密钥，通过 U 盘等工具将公钥（通常为`～/.ssh/id_rsa.pub`）拷贝到手机上。这里可能需要授予 Termux 存储权限，以让它读取密钥的内容；或者也可以选择用 Termux 打开文件 —— 该文件会被自动下载到 Termux 的 `～/downloads` 目录下。

   然后在手机端进入存储有公钥的目录，执行命令`cat id_rsa.pub > ~/.ssh/authorized_keys` 即可成功导入公钥。

3. 连接 SSH 服务

   现在在手机端运行`sshd &`开启 SSH 服务，在 PC 端执行`ssh -p 8022 <username>@<ip_addr>`即可连接上手机。其中的 username 可以通过在手机端执行`whoami`获取；ip 地址则可以通过`ip addr show`获取。

## 安装 Zsh

通常情况下安装 zsh 与 PC 端并无二致，如下：

```bash
apt-get install curl git zsh
sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

也有反映 oh-my-zsh 的官方一键安装脚本无法使用的情况，这时可以考虑使用 [oh-my-termux](https://github.com/4679/oh-my-termux) 。

{% note primary%}提示： 若出现 zsh 的 agnoster 主题（或其他依赖 powerline 字体的主题）无法正常显示，可将您的 powerline 字体拷贝到 `～/.termux/font.ttf` 后执行`termux-reload-settings` 。详见[知乎回答](https://www.zhihu.com/question/274678906)。{% endnote %}

## 更换 APT 国内源

通过以下命令修改 apt 源（这里使用清华源）：

```
export EDITOR=vim
apt edit-sources
# deb http://mirrors.tuna.tsinghua.edu.cn/termux stable main
```

记得在修改完后执行`apt-get update` 以更新 apt 列表。

## 开启存储权限

为了更方便的连接并管理手机内部存储，我们需要给 Termux 开启存储权限。通常开启存储权限后会在 home 目录下出现一个指向 sd 卡内部存储的 storage 文件夹；如果没有，则可以执行`ln -s /sdcard/ ~/storage` 将 sd 卡链接到 home 目录下。

## 使用 scp 传输文件

安装完 Termux 之后最常用的功能之一就是在手机与 PC 之间传输文件，这点可以使用`scp` 来实现。例如：

```bash
# PC to Phone
scp -P 8022 -r ~/Documents <username>@<ip_addr>:~
# Phone to PC
scp -p 8022 -r <username>@<ip_addr>:~/storage/Downloads ~
```

当然也可以使用`rsync` 等类似工具 。

## 小技巧及其他

- 或许你会在手机端打开 vim 却尴尬地发现没有 `ESC` 键。在 Termux 中，你需要用`VolUp + q` 来打开软键盘；类似的还有`VolUp + w|a|s|d` 来代替方向键。详细可以参阅 [Termux Wiki](https://wiki.termux.com/wiki/Touch_Keyboard) 。

- 你可以在你的 Shell 配置文件里配置一些别名来代替冗长的命令。例如我的方案：

  ```bash
  export IP_PHONE='<ip_addr>'
  alias ssh-phone='ssh -p 8022 <username>@$IP_PHONE'
  function ssh-up() {
  	if [ -n $1 ]; then
  		scp -P 8022 -r $1 u0_a177@$IP_PHONE:~
  	fi
  }
  function ssh-down() {
  	if [ -n $1 ]; then
  		scp -P 8022 -r u0_a177@$IP_PHONE:~/$1 ~
  	fi
  }
  ```

  这样就能用一些简单的命令来进行日常任务了。