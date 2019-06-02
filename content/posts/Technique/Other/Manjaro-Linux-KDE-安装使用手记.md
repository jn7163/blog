---
title: Manjaro Linux + KDE 安装使用手记
date: 2018-06-22 19:01:01
tags:
  - manjaro
---

“`Manjaro`是一款基于 Arch Linux、对用户友好的 Linux 发行版。” 有多友好呢？在与 Debian 苦战两天无果之后，我用短短半个小时就让 Manjaro 在我老旧的笔记本上跑了起来。再加上内容丰富的 Arch Wiki 使得我在遇到问题时都有据可循，我毫不犹豫地弃 Debian 转 Manjaro。

<!--more-->

## 安装 Manjaro
Manjaro 的安装过程简洁无比。完善的图形界面引导取代了 Arch 繁琐难懂的命令安装流程。这里就简单提几点需要注意的。

### 制作启动盘
首先需要下载一个 Manjaro ISO 镜像。可以考虑一下从 [Manjaro 中文站](https://www.manjaro.cn/) 下载。然后自行选择一种桌面环境，我个人选择的是较为美观的 KDE。当然如果有需要的话国内的深度桌面环境 DDE 也是可以的，不过需要自己手动安装。

其次就是刻录启动盘。官方推荐使用 [rufus](https://rufus.akeo.ie/) 来刻录启动盘。在刻录的时候记得选择`dd`模式刻录 ISO 文件，否则会出现问题。

[![](https://i.loli.net/2018/09/16/5b9e4f986cb9e.png)](https://i.loli.net/2018/09/16/5b9e4f986cb9e.png)

刻录完毕后将U盘插入电脑重启就可以进入安装环节了。注意要在启动选项中选择从U盘启动（对于不同品牌的电脑可能有不同的启动选项配置快捷键，通常为`F2`，`F10`，`F12`等）。

### 安装引导与初始设置
重启后进入安装界面，简单配置一下时区之类的本地化信息之后选择`Boot Manjaro`项就会从U盘启动安装引导系统。这个时候的系统也是完全可以用的，大家可以测测网络玩玩终端之类的。总之如果觉得这个系统还算令人满意，那你就可以双击桌面上的`Install Manjaro Linux`进入安装引导。由于图形界面十分完善，基本没有什么难点，大家只要跟着流程走就行了。

但是对于安装双系统的同学就需要小心了。引导程序也特意用红色标出——表示抹除磁盘是个较为危险的选项。如果你的电脑上原来就装了Windows，那选择抹除磁盘会让你的Windows直接被清除。这个时候对分区流程清楚的同学可以选择手动分区，嫌麻烦的同学呢则可以选择直接取代一个分区。例如我在下图中选的就是用Manjaro直接取代原来的D盘。

[![](https://i.loli.net/2018/09/16/5b9e501fdc0e7.png)](https://i.loli.net/2018/09/16/5b9e501fdc0e7.png)

然后基本就大功告成啦。

### 额外配置
安装完 Manjaro 后还需要进行一些小修补。打开命令行（`Alt + Space`输入`konsole`回车或者`F12`调出`Yakuake`）并执行以下命令：

```bash
sudo pacman-mirrors -g       # 排列源
sudo pacman-optimize && sync # 同步，固态硬盘不要执行
sudo pacman -Syyu            # 升级系统，可能耗时较长
```

有时候用 pacman 下载安装软件包会出现无法连接服务或下载速度很慢的现象，这时候可以考虑用官方镜像源。

```bash
sudo pacman-mirrors -i -c China -m rank # 排列并选择软件源
```

还需要配置 archlinuxcn 源，在这里你可以下载到一些国内更常用的软件（sogopinyin / netease-cloudmusic）。你需要修改`/etc/pacman.conf`文件并添加下列注释中的语句：

```bash
sudo vim /etc/pacman.conf
# [archlinuxcn]
# SigLevel = Never
# Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch
```

然后更新数据并导入签名：

```bash
sudo pacman -Syyu
sudo pacman -S archlinuxcn-keyring
```

嗯，这下真的大功告成了。让我们开始安装软件吧。

## 安装常用软件

*这个模块目测会不断更新。我会在这里把自己觉得实用的或者有意思的软件推荐给大家。*

*注意：接下来的许多内容可能只在 Arch 或 KDE 下有效。*

### 桌面环境美化
俗话说颜值是第一生产力（真的假的？）。但毋庸置疑一个漂亮的学习工作环境是会提高人的生产效率的。这里我就推荐一些桌面美化方案给大家参考。顺便晒一晒自己的桌面：

[![](https://i.loli.net/2018/09/16/5b9e51de18248.png)](https://i.loli.net/2018/09/16/5b9e51de18248.png)

#### Breezemite
`Breezemite` 是一款模仿 MacOS 窗口按钮的 KDE 按钮主题。可以在 *系统设置 => 应用程序风格 => 窗口装饰* 里获取新主题安装它。注意你也可以在按钮选项中把窗口按钮调到左上角。

#### Cairo-dock
顾名思义，所谓`cairo-dock` 就是模仿 MacOS 做的一个 dock，效果见屏幕底端。可以使用三维也可以平面化，包括特效、主题都可以在图形界面里设置，还是比较方便的。安装方法如下：

```bash
# 如果尚未安装 yaourt
# sudo pacman -S yaourt
yaourt -S cairo-dock cairo-dock-plug-ins
```

注意右键 dock 可以选择开机自启动。

#### Conky
`conky` 就是屏幕左侧的那个面板。使用`conky` 可以帮助你实时监测系统的一些有关信息，当然主要是看起来很酷[/捂脸]。在 Manjaro 上你可以直接一行命令安装：

```bash
sudo pacman -S conky   # 安装
conky -C               # 默认配置
conky -c ~/.conkyrc    # 用户设定配置文件路径
```

官方 [repo](https://github.com/brndnmtthws/conky) 里提供了不少用户配置文件（User Configs），大家可以下载下来直接使用。

#### Minimal Clock
KDE 一个鲜明的特色就是在它的桌面上一切都是组件化的。通常情况下你可以在自带的部件里找到很多好看有使用的部件，但在时钟这方面的表现不尽如人意，于是乎我就在 [store.kde.org](https://store.kde.org/) 上找到了 [Minimal Clock](https://store.kde.org/p/1173746/) 组件（效果见上图）。当然你也可以在这个网站上找到大量其他组件和主题来美化自己的桌面。

### Linux 终端体验
与 Windows 相比，Linux 拥有无比强大的终端。善用终端能大大提高你的生产效率。下面就介绍一些终端下好用的命令行工具。

#### htop
`htop` 是 Linux 原生 `top` 命令的升级版。它是一个终端下的交互式任务管理器。你可以使用 htop 直接在终端管理你的进程与任务。你可以直接运行 `sudo pacman -S htop` 来安装。

#### screenfetch
`screenfetch` 命令能在你的终端打印出你的系统信息，甚至图标（很酷不是吗）！同样可以使用 pacman 直接安装。当然你也可以考虑使用 `neofetch` 作为替代品。

#### ranger
`ranger` 是一款终端下的文件夹管理器，并且支持 vim 快捷键。善用它可以大量减少你在终端与 dolphin（KDE 默认图形界面文件夹管理器）之间的不必要切换。同样可以一行命令安装：`sudo pacman -S ranger` 。

#### thefuck
这个神奇的包是用 python 编写的，它的作用就是 —— 当你在终端输入了错误命令时，只要再输入`fuck` 就能自动纠错并执行。可以说是很符合人机工程学了[/笑哭]。安装方式为：`pip install thefuck` 。

#### vim & vimtutor
无需多说，vim 绝对是文本编辑界的神器。据说甚至有巨佬全 vim 开发，真是令我等瑟瑟发抖。不过话说回来，vim 也绝对是提效神器，免去了你在终端与其他文本编辑器之间切换的痛苦——再加上一定的插件，写起代码来都是毫无压力。安装命令如下：`sudo pacman -S vim`。

但 vim 的学习曲线也是无比陡峭，新手打开 vim 以后甚至可能都不会退出。这时候就需要 `vimtutor` 命令来帮助啦。如果你的 vim 正常安装成功，你就能在终端直接打开 vimtutor， 然后开始你的 vim 之旅吧！

#### w3m
`w3m` 是一款运行在终端界面下的网页浏览器。对的你没听错，就是浏览器。如果有必要的话它甚至可以在终端显示图片。对于我这样一个 Zen Mode 重度患者来说，写代码的时候退出 Zen Mode 切换浏览器是件很麻烦的事情，这时候结合 w3m 和 yakuake 我就能直接简单浏览网页。怎么下载？问 pacman 去。

#### yakuake
堪称 KDE 下的终端神器。如果你安装的是 Manjaro + KDE，那么 yakuake 已经预安装在你的系统里了，只要按`F12` 就能唤醒。其他 Linux 发行版的同学可以手动安装，一般在官方源里都有提供。

Gnome 或 Xfce 桌面的用户可以考虑使用 Tilda 或 Guake。

#### yaourt
Arch Linux 强大之处之一就在于它数量庞大的 AUR（Arch User Repositry），许多优秀的第三方软件都在这里被维护和分流。但 pacman 的官方软件源里并不包含 AUR，所以 yaourt 就此横空出世。yaourt 是 Yet AnOther User Repositry Tool 的缩写，通过 yaourt 我们可以极大地扩展 Arch 的能力。作为 Arch 延伸版的 Manjaro 自然也能使用 yaourt，安装流程如下：

```bash
sudo pacman -S yaourt
# 如果安装失败，可能需要添加第三方软件源
sudo vim /etc/pacman.conf
# [archlinuxfr]
# SigLevel = Never
# Server = http://repo.archlinux.fr/$arch
sudo pacman -Sy yaourt
```

>   提示： yaourt 使用时无需 sudo，否则会出错；如果频繁被提示需要调用 vim 等编辑器，可以考虑在`~/.zshrc` （bash 则为`~/.bashrc`）里添加 `export VISUAL="vim"` 。

可以参考 DigitalOcean 的这篇文章：[How To Use Yaourt to Easily Download Arch Linux Community Packages](https://www.digitalocean.com/community/tutorials/how-to-use-yaourt-to-easily-download-arch-linux-community-packages)。

>   注意：yaourt 目前已经停止维护，用户可以考虑迁移到 aurman 或 yay 。可以参考 [Arch Wiki](https://wiki.archlinux.org/index.php/AUR_Helpers) 详细了解。

#### zsh & oh-my-zsh
号称终极 Shell，拥有远超 bash 的高效功能。

当然，尽管 zsh 功能强大无比，但初期由于配置困难复杂，大多数人对其望而却步。直到有一天一位叫 Robby Russell 的程序员写出了 oh-my-zsh 这一 zsh 配置和插件管理器。可以参考官方网站 [ohmyz.sh](https://ohmyz.sh/) 。这里贴出 Arch 下的安装方法：

```bash
sudo pacman -S zsh               # 安装zsh
chsh -s $(which zsh)             # 修改默认shell
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"    # 使用curl安装oh-my-zsh
```

可以考虑安装一些主题和插件，例如：

```bash
vim ～/.zshrc
# ZSH_THEME="agnoster"
# plugins=(
#     autojump
#     extract
#     git
#     zsh-syntax-highlighting
# )
```

其中`autojump` 和 `zsh-syntax-highlighting` 需要手动安装：

```bash
# 安装 autojump
sudo pacman -S autojump
# 安装 zsh-syntax-highlighting
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
zsh
```

安装配置就此大功告成啦！

### 其他优秀软件
#### google-chrome

chrome 浏览器的 AUR 版本，可以从用 `yaourt -S google-chrome` 安装。

#### fcitx
fcitx 是 Free Chinese Input Toy for X 的缩写，国内也常称作小企鹅输入法，是一款 Linux 下的中文输入法，支持 pacman 安装：`sudo pacman -S fcitx` 。这里提一下几点比较常见的：

安装完 fcitx 之后还不能直接使用，需要编辑配置文件：

```bash
vim ~/.xprofile
# export GTK_IM_MODULE=fcitx
# export QT_IM_MODULE=fcitx
# export XMODIFIERS="@im=fcitx"
```

然后重启 fcitx 服务即可使用。

- 建议安装 fcitx-googlepinyin 和 fcitx-configtool。前者提供更友善的输入联想，后者提供图形界面配置工具。

- 建议安装 fcitx-cloudpinyin 的大陆用户打开 fcitx-configtool 在附加组件里将云拼音来源改为百度，因为 Google 的服务在国内不稳定。

- 如果发现在命令行等界面下无法调出输入法，可以尝试安装 fcitx-qt5（针对 KDE 用户）。详细情况可以参见 Arch Wiki 。

#### netease-cloud-music
不用说，国内主流音乐播放器中唯一一款支持 Linux 的。操作起来和 Windows 下的几乎没有区别。虽然个人不是网易云音乐粉，但还是要在这里感谢下团队的对 Linux 用户的付出。可以从 AUR 安装：`yaourt -S netease-cloud-music` 。

#### shadowsocks-libev
某科学上网工具。在这里就不做详细介绍了，可以直接参见官方仓库。Arch 上的安装也很简单，直接使用 pacman 即可：`sudo pacman -S shadowsocks-libev` 。安装完毕后可以用 `ss-local` 命令搭建起本地服务器，用户以本地服务器为代理链接远端服务器从而访问 Google、Youtube 等网站，详细可以参考 [Arch Wiki][1] 。这里提几点小建议：

- 浏览器端可以考虑安装 [switchyomega](https://www.switchyomega.com/) 插件，它可以帮你自动切换代理节省流量。

- 终端代理可以考虑安装 proxychain-ng，同样可以直接通过 `sudo pacman -S proxychains-ng` 安装。然后修改配置：

```bash
sudo vim /etc/proxychains.conf
# socks4 127.0.0.1 9050 （×）
# socks5 127.0.0.1 1080 （√）
```

使用时在需要代理的命令前加 proxychains4 就可以了，例如：`proxychains4 ping www.google.com` 。

- 图形界面管理器 [electron-ssr](https://github.com/erguotou520/electron-ssr)。Windows 下的 shadowsocks 客户端具有图形界面，而 Linux 下却只能用命令还要手写配置，这是比较烦人的。原先也有一个 shadowsocks-qt5，不过现在早就寿终正寝了。electron-ssr 总体表现十分优异，和 Windows 的客户端基本一致，在加上 Electron 天生跨平台→_→。安装也比较简单：`yaourt -S electron-ssr` 即可。

#### typora
一款非常优秀的 Markdown 编辑器，也是我目前写博客（包括这篇文章）使用的主力编辑器。界面简介而功能强大，基于 Electron 强势跨平台，推荐使用。可以从 AUR 安装：`yaourt -S typora` 。

#### visual-studio-code
Microsoft 最新作品，基于 Electron 完全跨平台，个人体验非常优秀的一款编辑器，社区活跃插件众多。话说 Microsoft 这几年来开源化浪潮是愈演愈烈，希望能一直做下去。还是从 AUR 安装：`yaourt -S visual-studio-code-bin` 。

#### wireshark-qt
Wireshark 是一款免费开源的包分析器，可用于网络排错、网络分析、软件和通讯协议开发以及教学等。然而不幸的是官方网站上仅提供了 Windows 和 MacOS 的版本。不过不用担心，你依然可以使用 pacman 直接从 Arch 的 community 软件源里直接安装 wireshark-qt 。wireshark-qt 就是用 qt 做成的 wireshark 前端界面，它依赖于终端版的 wireshark-cli 。

这里提示一点： 通常直接安装完后启动 wireshark 会提示 /usr/bin/dumpcap 无权限，有些用户会因此选择在命令行里`sudo` 打开 wireshark，这其实很不安全。正确做法是运行命令`gpasswd -a username wireshark` 然后重新启动，详情可以参见 [Arch Wiki][2] 。

## 常见配置问题
### Dolphin 设置
Dolphin 是 KDE 下默认的文件管理器，整体来说做的很不错，但可能有一些使用不太令人习惯（例如单击直接打开文件）。这里列出几点优化配置建议：

- 双击打开文件（夹）：这个设置选项深藏在与 dolphin 毫无关系的角落里[/无奈] 。打开系统设置=>桌面行为=>工作空间=>点击行为可以看见这一选项。
- 也有人可能会发现在Dolphin中使用 Del 键是很危险的：它会在没有任何确认的情况下直接删除你的文件。可以在配置Dolphin=>常规=>确认中打开文件删除确认。

### Grub 设置
在默认情况下我们打开 grub 引导菜单以后只有5秒钟的时间选择系统，这个能会带来些许不便。可以通过以下命令来修改 grub 配置：

```bash
sudo vim /etc/default/grub   # 修改为 GRUB_TIMEOUT=30
sudo update-grub             # 更新 grub 配置
```

### 系统设置
#### Home 目录下的中文名文件夹
如果在安装时选择了中文环境，那么系统就会自动生成一些本地化的文件夹，例如下载、桌面等。但是在很多情况下使用中文名文件夹很不方便，可以在*应用程序=>位置*里修改。

#### 开机自启动程序
很多时候我们需要使一些程序开机自行启动，这时候可以在*开机和关机=>自动启动*里设置，可以添加应用程序也可以添加脚本。当然也可以直接把可执行文件复制到`～/.config/autostart` 目录下。

#### 文件格式与应用程序关联
有时候我们需要针对不同的文件采用不同的程序打开，而默认程序往往不尽如人意。这时候可以在*应用程序=>文件关联*里修改。当然也可以在 Dolphin 下右键属性直接更改某一格式的默认打开程序。

#### 修改默认浏览器
包括但不限于默认浏览器，还有例如文件夹管理器等。可以在*应用程序=>默认程序*里修改。



[1]: https://wiki.archlinux.org/index.php/Shadowsocks_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)
[2]: https://wiki.archlinux.org/index.php/Wireshark_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)

