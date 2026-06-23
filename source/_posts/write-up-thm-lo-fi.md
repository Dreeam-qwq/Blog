---
title: 'Write-up: Lo-Fi'
categories:
  - 网络安全
tags:
  - TryHackMe
  - Web
date: 2026-06-23 17:03:33
---


## 0x0 前言

TryHackMe 把这个房间标为 5 minute hacks，确实简单，5 mins 就够了。

## 0x1 步骤

首先阅读给的信息，知道了这个房间是关于 Local file inclusion 和 Path traversal 两个漏洞相关。

因此就可以 LFI 路径遍历入手。网上搜索可以知道，LFI 路径遍历漏洞是利用各种方法来滥用输入的文件路径，来读取或者执行预期目录之外的东西。

![](/img/write-up/writeup-lo-fi-1.png)

通过观察页面，发现其他页面的跳转是直接引用目标页面的 PHP 文件（相对路径）。那么漏洞的利用方式就是构造自己的文件路径，来访问系统上的其他文件。这时候也暴露了这个网站使用的是 PHP 编写的。

使用 [Local File Inclusion Cheat Sheet](https://www.vulnsy.com/cheat-sheets/lfi), 可以知道常见的 LFI 路径遍历的 payload 都有哪些，随便挑一个出来试试，如下：

`../../../../../../../../etc/passwd`

大多数文件系统都会折叠过多的 `../` 路径，因此可以添加非常多的 `../` 来绕过一些检测规则，并尝试访问系统上的文件。

拼凑进 URL：
`http://IP/?page=../../../../../../../../etc/passwd`

![](/img/write-up/writeup-lo-fi-2.png)

那么验证了这种漏洞利用的方法确实可行，就可以找 flag 了。

根据所给信息，flag 位于文件系统根目录。

鉴于 THM 上很多其他房间的 flag，如果是以文件形式存在的，都位于 `flag.txt` 中。所以我们可以大胆的猜测本房间的 flag 位于 `/flag.txt`

![](/img/write-up/writeup-lo-fi-3.png)

试一下，flag 就出来了。

## 0x2 结语

非常简单的一个房间，还没有涉及到 `php://filter` 的一些高级玩法，或者使用各种 payload 变体来绕过规则检测。

同时 [Local File Inclusion Cheat Sheet](https://www.vulnsy.com/cheat-sheets/lfi) 内提供的 payload 有原理的解释，并且附上了 payload 的来源，都非常具有学习的价值，推荐一并查看。
