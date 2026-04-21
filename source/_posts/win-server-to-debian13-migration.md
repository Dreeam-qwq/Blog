---
title: 记服务器迁移：从 Windows 迁移至 Debian 13
categories:
  - 系统运维
date: 2026-04-21 03:55:20
tags:
---


## 为什么迁移到 Linux 上？

- 更多的可用内存

  相比于 Windows，Linux 系统自身的占用内存极低。迁移后的实际体验也是非常香，空载情况下几乎可以做到只有 1GB 左右的占用。剩余充足的空闲内存可以给 MC 服分更多的内存，甚至还可以再部署 MySQL 和 PostgreSQL 数据库。

- 更好的网络相关的优化

   Linux 系统原生支持 TFO 优化（TCP Fast Open）可供 Netty 使用。并且 Netty 可以选择 Epoll 或者 io_uring 作为网络的异步 I/O 模型，后者 io_uring 可以提供更好的网络吞吐性能。这些都是在 Windows 上享受不到的。Windows 上没有 TFO，Netty 的网络 I/O 模型也只能用 Nio。

- 支持集成 async profiler

  支持使用 [async profiler](https://www.baeldung-cn.com/java-async-profiler) 作为 Spark 的分析器，以提供更准确的分析结果，并且不会造成很大开销，以至于影响 MC 服的 MSPT 和 TPS。

- 练手，玩

  刚好这学期有 Linux 的系统运维课，当然不能用学校 Lab 的服务器 VM 整自己的东西啦（ \
  趁着这次给自己的 MC 服维护，刚好把系统换成 Debian 来练手。可以享受更高的性能，也算是一种尝鲜的体验，毕竟以前还没自己用过非桌面环境的 Linux 环境（

## 0x0 迁移的准备

迁移前，首先简单介绍一下我的环境。这是一个在剑客云上的 VPS，EPYC7443 8vCPUs 18G 内存，有一个系统盘，和一个数据盘，系统是 Windows Server 2019。上面放着我自己运营的 Minecraft 服，包括跨服端，登录服，主服，还有一些杂七杂八的个人自用东西，比如测试 bug 用的测试服，跑性能对比测试用的 JMH 等等。

此次迁移的目标是换到最新的 Debian 13，并重新配置环境，迁移数据。

因为 VPS 面板提供的 Linux 系统镜像只有 Debian 12，我打算先重装到 Debian 12 随后升级到 13。

数据将会全盘格式化，包括数据盘。因为要使用 Linux 原生的 ext4 文件系统而不是 NTFS 系统，以获得更好的性能。这样做可以避免 ntfs-3g 带来的 FUSE 开销，和从 Windows NTFS 文件系统上遗留下来的不必要的权限映射问题，没错我是极端优化爱好者 www。

简单理了一遍现在系统上的数据，发现一些软件和工具可以等系统重装后，重新下载对应系统的版本并配置。因此只需要打包重要的数据，如 MC 服务端，个人文件夹等。最后理出来 ～200GB 的打包数据，二次测试了压缩包，确认所有数据没问题之后，上传备份至阿里云盘。

因机子的带宽有限，经过漫长的等待，备份的平均上传速度为 ～400～700KB/s，上传时间 ～5 天，个人感觉还算可以～

## 0x1 系统配置

备份上传完毕，可以正式开始迁移工作了。

首先使用 VPS 面板提供的重装系统功能，“一键重装”至 Debian 12。不保留数据，因此选择：格式化全盘。

稍等片刻，待重装任务执行完毕，就可以连接 SSH 开始重新配置环境了。

此次迁移中的整个重新配置过程，以及后续的运维，都会使用 VS Code，配合 Remote SSH 扩展进行远程操作，大大简化运维难度。

使用这套方案，可以充分利用 VS Code 文本编辑器本身提供的功能，进行快捷的文本编辑，目录浏览，并且能够语法高亮配置文件，日志文件等，和在自己电脑上写代码没啥区别；用 VS Code 的终端作为 SSH 的终端，与远程服务器进行命令交互。

同时，VS Code + Remote SSH 也支持自动识别并转发目标服务器上的内网服务至本机，或者手动添加服务的端口号进行转发。

VS Code 集成的端口转发是一个非常强大且方便运维的功能，本质上是通过 SSH 的加密连接，将远程端口映射到本地。因此无需将所有服务暴露到公网，在连接了 SSH 的情况下，就能够在本地轻松，安全的访问远程服务了。

比如，在本地浏览器访问目标服务器内网的 Web 服务，在本地进入目标服务器上未暴露在公网的 MC 测试服等等。

![](/img/win-to-debian13/vscode-ssh-port-forwarding.png)

推荐自行网上搜索如何配置、使用 Remote SSH 扩展，已经有很多非常详细的教程了，相关步骤就不在本文中展开了。

### 设置 hostname

成功连接 SSH 之后，先设置一个自己 VPS 的 hostname。

这样做更规范，可以更好的标识你的云服务器，在终端命令行里的显示也会更好看。

当然其他操作下也会用到 hostname，比如添加软件源的 GPG key 时，使用 sudo 命令会触发对本地 DNS 解析。如果没有配置，会报错提示无法解析当前的 host。

排查发现，当前 Debian 系统的 hostname 为 `ECS1150`，但是 `/etc/hosts` 里的本地解析为 `127.0.0.1 debian`，hostname 并不一致。这可能也解释了为什么 sudo 命令触发的本地解析会报错。

在这里我使用了 `dreeam-server` 作为自己 VPS 的 hostname，你可以改成自己喜欢的名字。

```sh
sudo hostnamectl set-hostname dreeam-server
```

设置完之后，修改 `hosts` 文件，添加自己的 hostname，作为本地 IP 的解析。

```sh
code /etc/hosts
```

把对 IP `127.0.0.1` 的解析改为 `127.0.0.1 [your-hostname]`，括号内替换成你的 hostname。

![](/img/win-to-debian13/hosts-config.png)

### apt 软件源配置

拿到 Linux 系统终端后做的几件事之一，当然是 apt update 然后 upgrade 啦。

我在使用 `apt update` 时，遇到了如下的提示。

![](/img/win-to-debian13/wrong-apt-source.png)

![](/img/win-to-debian13/sources-list-before-edit.png)

这说明 `sources.list` 是有问题的。目前 `sources.list` 内的软件源还是指向的 Debian 系统的 ISO 镜像。我猜测应该是服务商配置预装系统镜像不规范导致的。

打开 `/etc/apt/sources.list` 将里面的内容覆盖成 Debian 的官方 apt 源，如下所示。

```sh
deb https://deb.debian.org/debian bookworm main contrib non-free non-free-firmware
deb https://deb.debian.org/debian bookworm-updates main contrib non-free non-free-firmware
deb https://security.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware
```

（当然你可以换成国内的镜像源，当时一时间没想到用镜像源。。）

保存退出。随后 `apt update` 和其他 apt 命令就可以正常使用了。

### 安装基础工具

我们先使用以下命令安装几乎必备的命令和基础工具，在后续的配置和运维中都是必不可少的。

```sh
# 三个非常常用的命令
apt install sudo
apt install curl
apt install net-tools
#（必需）7z 解压缩工具
apt install p7zip-full
#（必需）一些软件安装时需要用到解压缩
apt install unzip
apt install zip
# 以下两个可选
apt install git
apt install gnupg2
```

### 升级至 Debian 13

相比于 12，Debian 13 包含了很多安全性修复，并更新了很多软件包的版本，也加入了一些高雅的新特性。对于喜欢追求现代和最新版的我来说，非常值得一试。

考虑到因为是新系统，未安装新的东西，也没有现有的数据。因此可以直接安全更新到 Debian 13，无需任何备份。

首先在 Debian 12 下，更新所有软件包至最新版。简单运行 `apt update` 和 `apt upgrade` 命令更新完之后，将 `/etc/apt/sources.list` 内的 apt 软件源指向 Debian 13 的源。

以下是 Debian 13 的官方源：

（可以看到 URL 后面跟着的词变了。Debian 12 的 代号是“bookworm”，Debian13 的是“trixie”。）

```sh
deb https://deb.debian.org/debian/ trixie main non-free contrib non-free-firmware
deb https://security.debian.org/debian-security trixie-security main non-free contrib non-free-firmware
deb https://deb.debian.org/debian/ trixie-updates main non-free contrib non-free-firmware
```

但是这里我们可以直接用国内的镜像源，在国内获得更快的下载速度。我选择了清华大学的镜像源，服务更稳定，提供的版本也更全。

复制以下内容，直接粘贴替换 `sources.list` 里旧的源即可。

```sh
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ trixie main non-free contrib non-free-firmware
deb https://mirrors.tuna.tsinghua.edu.cn/debian-security trixie-security main non-free contrib non-free-firmware
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ trixie-updates main non-free contrib non-free-firmware
```

保存退出。再次执行以下命令，将系统完全升级至 Debian 13。

```sh
apt update
apt upgrade
apt full-upgrade
```

随后使用 `reboot` 重启服务器。

### 挂载数据盘

如果服务商提供的重装系统功能已经完成了挂载数据盘，格式化并且不需要修改挂载点，可以直接略过本步骤。

Linux 的根目录下有很多系统自带的目录，每个目录都有其对应的用途。

常见目录如下：

- /dev - 设备目录，如储存设备
- /etc - 配置文件目录
- /mnt - 临时挂载点，也可以用于长久的挂载路径
- /opt - 常用的软件安装目录
- /srv - 系统或者应用服务的数据的目录，比如存放网站或者 FTP 服务器的数据
- 。。。

根据各个根目录文件夹本身的语义和最佳实践，`/mnt` `/srv` 这些路径都可以作为数据盘的挂载点。只要符合个人或者团队的习惯，并且方便运维，你也可以选择自己喜欢的路径，比如 `/data` 等等。

最后我选择了 `/mnt/data` 作为我 VPS 数据盘的挂载路径。

#### 配置挂载点

使用以下命令查询分区信息并创建挂载点的目录：

```sh
lsblk
mkdir -p /mnt/data
```

如下图所示，使用 `lsblk` 发现数据盘 `/opt/1` 被挂载到 `/dev/sdb` 整个磁盘上了，而不是 sdb1 分区上。这并不是一个很规范的操作，因此得先取消这个挂载，再将 sdb1 分区挂载到我们的目录 `/mnt/data`。

![](/img/win-to-debian13/partition-info-before.png)

首先使用 `umount` 取消这个挂载。

```sh
umount /opt/1
```

可以使用 `lsblk` 再次验证是否取消挂载成功。

然后就是格式化分区并重新挂载了。

```sh
# 格式化分区
mkfs.ext4 /dev/sdb1
# 挂载到 /mnt/data 路径下
mount /dev/sdb1 /mnt/data
```

#### 开启自动挂载

如果这时候重启，配置的挂载就会失效，因此得配置开机自动挂载。

首先获取 sdb1 分区的 UUID 唯一标识符：

```sh
# 获取分区 UUID
blkid /dev/sdb1
```

知道了分区的 UUID，就可以编辑 `/etc/fstab`，将分区信息加入开机自动挂载配置。

![](./img/win-to-debian13/auto-mount-on-start-config-before.png)

配置如下，把括号内的 `disk-uuid` 换成实际的分区 UUID

```sh
UUID=[disk-uuid] /mnt/data  ext4  defaults,nofail  0  2
```

在这行配置中，这些字段按顺序分别为：

1. 分区 UUID，对应需要挂载的分区。
2. 挂载点的路径。
3. 分区的文件系统为 `ext4`。
4. 挂载的选项。
  `nofail` 表示如果在启动时，找不到对应分区设备不会报错。这可以避免阻止系统启动，导致无法远程连接服务器排查问题。
  可以查看 [mount 的 man 手册](https://www.man7.org/linux/man-pages/man8/mount.8.html) 来了解所有的选项。
5. 是否要备份文件系统，设为 `0` 默认。
6. 启动时检查文件系统的顺序，非根目录文件系统，设为 `2` 默认。

配置后如下：

![](./img/win-to-debian13/auto-mount-on-start-config-after.png)

保存配置之后。重新挂载一遍，确认开机自动挂载配置成功，并且没有报错。

```sh
# 测试是否挂载成功
umount /mnt/data
# 挂载所有 /etc/fstab 内定义的分区
mount -a
```

#### 错误排查

##### 1. 分区缺失

如果使用 `lsblk` 查询分区信息，发现只存在 `/dev/sdb` 磁盘，但是 `/dev/sdb1` 分区不存在。

这时候需要使用 `fdisk` 命令编辑磁盘，自己创建一个新的分区，否则可以直接格式化。

```sh
sudo fdisk /dev/sdb
```

进入磁盘编辑之后，可以参考以下步骤。

1. 输入 `n`，选择“创建新分区”
2. 输入 `p`，选择分区类型为“主分区”
3. 输入 `1`，选择分区编号，使用默认值 `1`
4. 回车，定义扇区起始位置，使用默认值 `2048`
5. 回车，定义扇区结束位置，默认使用最后一个扇区，也就是使用整个数据盘空间。
6. 输入 `w`，保存，将新分区写入磁盘

完整步骤如图所示：

![](/img/win-to-debian13/partition-creation-steps.png)

##### 2. 未格式化导致挂载失败

在挂载分区到挂载点时，遇到了如下报错：

```sh
root@ECS1150:~# mount /dev/sdb1 /mnt/data
mount: /mnt/data: wrong fs type, bad option, bad superblock on /dev/sdb1, missing codepage or helper program, or other error.
       dmesg(1) may have more information after failed mount system call.
```

很明显挂载失败了。我们可以使用 `blkid` 命令来查询 `/dev/sdb1` 分区的属性信息。

```sh
root@ECS1150:~# blkid /dev/sdb1
/dev/sdb1: PARTUUID="5cb34ce7-01"
```

可以看到，有分区的 UUID，但是没有其他分区属性来。这说明磁盘没格式化，得格式化。

## 0x2 安装必备工具

### 1panel

[1panel](https://1panel.cn/) 是一个基于 Web 可视化界面的 Linux 管理面板，允许你更方便的运维你的 Linux 服务器，和管理数据库等相关的服务。

你可以查看 [1panel 的 官方文档](https://1panel.cn/docs/v2/installation/online_installation/) 以了解如何使用一键安装脚本部署，和后续的配置。

我实际上并没有用到 1panel 的很多功能。单纯是将它暴露在公网下，为了更好的查看系统状态，和文件管理，不需要每次都连 SSH 来看东西了。

### MCSManager

在这里，我使用 [MCSM](https://www.mcsmanager.com/) 作为 MC 服务器实例的管理，和后台服务持久化。

对于多子服的服务器，使用 MCSM 作为管理方案可以更方便的查看、管理各子服的终端，以达到和 Windows cmd 开服相似的效果。这样也可以避免使用 `screen` 或者 `nohup` 来“手动”管理后台服务和切换终端会话了。

你可以查看 [MCSM 的部署教程](https://docs.mcsmanager.com/zh_cn/)，使用一键安装脚本进行部署，并了解后续的相关配置。

## 0x3 部署主要服务

### Docker

使用 Docker 可以隔离各服务的资源，提升安全性，避免服务直接访问系统，还可以更方便的部署服务。

当然套一层 Docker 容器和直接跑在系统上相比，始终是有一点轻微开销的。

目前的架构足够简单，我决定不采用 Docker 来隔离数据库等服务。如果你追求极致的安全，可以考虑使用 Docker。

### 备份下载与部署

阿里云盘官方没有提供 Linux 版，因此我选择了社区的命令行工具 [aliyunpan](https://github.com/tickstep/aliyunpan)。

推荐阅读 aliyunpan 的 [README](https://github.com/tickstep/aliyunpan/blob/main/README.md) 和 [命令手册](https://github.com/tickstep/aliyunpan/blob/main/README.md) 来了解如何使用。

考虑到我的备份文件基本是百 GB 级的大文件，因此使用了命令手册中的 Linux 后台下载脚本，并配合以下命令：

```sh
nohup ./download.sh > aliyunpan.log 2>&1 &
```

这个命令使用 `nohup` 将下载挂到后台进程中进行，并将日志输出重定向到到 `aliyunpan.log` 文件。这样可以防止因 SSH 连接中断导致直接下载的任务失败，并且可以在下载任务完成后轻松审计下载是否成功，文件完整性校验是否通过。

备份下载完毕，接下来就是解压，并且把解压出来的东西用 1panel 移到对应的目录啦。然后在 MCSM 里重新创建各个子服的实例，指向对应的目录。

对于其他命令行程序，比如 QQ Bot 需要用到 [Napcat](https://napneko.github.io/guide/napcat)，MC 服和 QQ 的互联方案 [FlowGate](https://fgate.crashvibe.cn/)。我选择一起丢到 MCSM 进行管理，和后台常驻的实现。

方便快捷，就不用自己再手写 systemctl daemon 服务来实现开机自启和后台常驻了。

### 数据库安装

在之前的 Windows 环境中，我 MC 服所有插件的数据储存均使用 SQLite 或者文件储存，并没有用到任何关系型数据库。

可是随着玩家数据日益增大，我确实是有迁移到 MariaDB 或者 PostgreSQL 的打算，以获得更好的数据库查询性能。

但是截止撰写本文，我并没有来得及进行数据库的详细配置和相关数据的完整迁移。

目前只完成了数据库的安装，可能以后会写另一篇文章介绍数据库相关的东西，说不定呢？。

## 0x4 美化 & 自定义

一些美化和自定义的配置，方便日常的运维。

### 安装 Zsh

安装 Zsh 并设为默认 shell。

Zsh 提供极高的自定义，支持安装各种插件，主题以实现各种强大的扩展功能。

```sh
apt install zsh
# 设置为默认
chsh -s $(which zsh)
```

### Zsh 美化

推荐参考我好盆友的文章： [Zsh 终端的配置](https://ling.crashvibe.cn/posts/default/server-migration-with-rsync-and-zsh#6__zsh-%E7%BB%88%E7%AB%AF%E7%9A%84%E9%85%8D%E7%BD%AE)，对你的 Zsh 终端进行基本的美化和功能扩展，以提升运维效率和体验。

如你需要对 Zsh 进行深度的自定义定制，可以阅读 [从零构建 Zsh 环境](https://ling.crashvibe.cn/posts/tool/zsh-setup-guide)。

基于以上“Zsh 终端的配置”文章内提供的 `.zshrc` 配置，我进行了一点修改，使用 `zsh-users/zsh-syntax-highlighting` 语法高亮插件，以避免切换历史命令时出现爆栈的问题。

```sh
# zsh 套件四天王
zinit light zsh-users/zsh-completions
zinit light zsh-users/zsh-autosuggestions
zinit light zsh-users/zsh-history-substring-search
zinit light zsh-users/zsh-syntax-highlighting
zinit light romkatv/powerlevel10k

# Oh My Zsh 功能
zinit snippet OMZ::lib/completion.zsh
zinit snippet OMZ::lib/history.zsh
zinit snippet OMZ::lib/key-bindings.zsh
zinit snippet OMZ::lib/theme-and-appearance.zsh

# key binding
bindkey '^[[A' history-substring-search-up
bindkey '^[[B' history-substring-search-down
bindkey ',' autosuggest-accept

# 其他
zinit load djui/alias-tips

# To customize prompt, run `p10k configure` or edit ~/.p10k.zsh.
[[ ! -f ~/.p10k.zsh ]] || source ~/.p10k.zsh
```

保存关闭。重新进入 Zsh 之后，使用命令：

```sh
p10k configure
```

进行 p10k 主题的配置。随后就可以在终端看到修改后的终端样式了。

### 安装 fastfetch

[fastfetch](https://github.com/fastfetch-cli/fastfetch) 是一个实用工具，能够以带有系统 Logo 的美观终端界面显示各类基本系统信息。

我选择 fastfetch 作为 [neofetch](https://github.com/dylanaraps/neofetch) 的替代品，neofetch 在各种意义上都过时了，它的 GitHub 仓库已在 2024 年存档。

![](/img/win-to-debian13/fastfetch.png)

使用如下命令安装：

```sh
apt install fastfetch
```

### 安装 btop

[btop](https://github.com/aristocratos/btop) 是一个以高雅可视化界面显示当前系统状态和资源占用的监控工具，可以作为 `tpp`、`htop` 的替代品。

这里推荐阅读 [监控系统资源的最佳工具：btop](https://ling.crashvibe.cn/posts/tool/btop-monitoring-tool)，和 btop 的 [GitHub 仓库](https://github.com/aristocratos/btop) 来了解 btop。

![](/img/win-to-debian13/btop.png)

使用如下命令安装：

```sh
apt install btop
```

### 现代化 apt 软件源配置

Debian 13 引入了一个新特性，使用了新的软件源配置文件的格式，遵循 DEB822 格式，让源配置的内容更清晰可读并且更规范。旧的 `sources.list` 格式已被标注为弃用。

可使用以下命令一键将旧的 `.list` 源配置文件迁移到新格式：

```sh
sudo apt modernize-sources
```

迁移之后，可以看到原先的 `.list` 软件源配置已被移至 `/etc/apt/sources.list.d` 目录并且做了备份。

`sources.list` 文件也已变为新目录下的 `debian.sources`。

![](/img/win-to-debian13/modern-apt-source-dir.png)

新格式例子（MariaDB 12.2 的软件源配置）：

```yaml
# MariaDB 12.2 repository list - created 2026-04-18 09:03 UTC
# https://mariadb.org/download/
X-Repolib-Name: MariaDB
Types: deb
# deb.mariadb.org is a dynamic mirror if your preferred mirror goes offline. See https://mariadb.org/mirrorbits/ for details.
# URIs: https://deb.mariadb.org/12.2/debian
URIs: https://mirrors.tuna.tsinghua.edu.cn/mariadb/repo/12.2/debian
Suites: trixie
Components: main
Signed-By: /etc/apt/keyrings/mariadb-keyring.pgp
```

### 设置系统语言为中文

在使用 fastfetch 的时候，我发现当前的系统语言为 `en_HK:en`。

这很奇怪，这个不常见的语言会导致一些 MC 插件的警告报错，比如 floodgate，无法识别这个语言导致无法匹配正确的翻译文本，所以最好是换成中文 `zh_CN`。

使用以下命令设置系统语言：

```sh
sudo apt update
sudo apt install locales
sudo dpkg-reconfigure locales
```

用方向键选择到 `zh_CN.UTF-8 UTF-8` 之后，用空格键选择，随后选择 `OK`。

随后在接下来的界面中，将 `zh_CN.UTF-8 UTF-8` 设为环境默认语言。

设置完毕。你可以看到包括 apt 命令、其他系统输出的语言都已变为中文。

## 0x5 The End

至此迁移工作完美结束。

整个迁移主要使用 VS Code + Remote SSH 扩展进行操作，编辑配置。

在后续的运维中，为了提升运维效率。我目前采用了以下策略：

- 使用 1panel 进行文件上传/远程下载，批量删除或者移动文件等操作。
- 使用 MCSM 管理 MC 服，Bot 的实例和终端管理。
- 使用 VS Code 进行 SSH 连接，文件配置的编辑和查看。

## 0x6 踩坑总结

在 Linux 运维的过程中，如果你像本文一样，使用 VS Code 的 Remote SSH 来管理服务器或管理文件。请务必谨慎用它来操作文件，特别是删除/批量删除的文件操作。

VS Code 远程的文件选择是不可信的。

使用 VS Code 删除文件时，如果你同时进行多个删除操作，比如前一个文件删除操作尚未完成，就开始下一个。VS Code 的文件选择极有可能会将下一个的选中的文件“fallback"到目录下面的其他文件或者文件夹，导致文件被误删无法找回。

因为 Linux 系统默认没有文件的删除记录，或者类似文件回收站的，这类误删操作往往很难被发现。我的整个 workspace 文件夹也是因为这个误操作被删除了三次，才排查出来是这个问题（

个人建议还是使用 1panel 的文件管理来进行文件删除的操作。如果你更偏好使用命令操作，可以使用 `rm` 命令手工删除目标文件或文件夹。

## 一些参考

- https://blog.sleepstars.net/archives/debian-13-deb822-sources
- https://cn.console-linux.com/?p=11046
- https://itsfoss.com/display-linux-logo-in-ascii/
- https://linuxconfig.org/configuring-apt-sources-list-a-quick-reference-guide-for-debian-systems
- https://www.geeksforgeeks.org/linux-unix/linux-directory-structure/
- https://www.mintimate.cn/2024/08/03/neoFetchStyle/
- https://www.sysgeek.cn/debian-13-released/
- https://www.sysgeek.cn/upgrade-debian-13/
