---
layout: post
title: "How many different in curl and wget"
permalink: how-many-different-in-curl-and-wget
date: 2016-01-19 22:35:24
comments: true
description: How many different in curl and wget
keywords: curl wget
categories: linux command

tags: linux command curl wget
---

`wget`和`curl`是我们常用的linux命名，`wget`通常用来通过shell下载文件，而`curl`通常用于通过
Shell请求Web资源。两个使用场景有些不同和雷同， 但是区别和雷同具体是什么，还是需要仔细研究下两个命令的特点。

#### 故事起源:
最近做运维比较多，尤其使用docker，偶尔有需求在container里面做设置，当有需要在container下载文件时，发现没有常用的`wget`，如果在container里面使用，需要安装`wget`，瞬间傻眼，这肯定不是Best Practice，于是使用`curl`。

#### 故事结果：
屡次使用`curl`都是取回Page或者Error Page， 结果发现需要带上option`-O`

#### 对比`curl`和`wget`

**Both**--比如以下特性都是在`wget`和`curl`都同时支持的:

- 都可以通过`ftp`或者`http`下载文件。
- 都可以发送*HTTP POST* request。
- 都支持*HTTP cookies*
- 都支持*metalink*

> 以下则是区别，根据这些区别我们也可用更好的了解何时是Best Practice: 以下则是区别，根据这些区别我们也可用更好的了解何时是Best Practice:


**Differ**--以下则是区别，根据这些区别我们也可用更好的了解何时是Best Practice:

- `curl`是一个*library*，而`wget`是一个*command line*。
- `curl`支持非常多的协议，但是`wget`是不支持的，例如：`DICT, FILE, FTP, FTPS, Gopher, HTTP, HTTPS, IMAP, IMAPS, LDAP, LDAPS, POP3, POP3S, RTMP, RTSP, SCP, SFTP, SMB, SMTP, SMTPS, Telnet and TFTP`
- `curl`有提供API，但是`wget`则没有
- `wget`支持一个叫**Recursive download**的方式，可以带参数`depth`, default是5，可以深度递归抓取的内容。

##### Example:
拿下来文件为例，如果计划使用`curl`
```bash
curl https://octodex.github.com/images/welcometocat.png
```
以上只能在屏幕输出访问资源的内容。

所以需要带上参数`-O`, 才是下载文件
{% highlight bash %}
curl -O https://octodex.github.com/images/welcometocat.png
{% endhighlight %}

然而如果使用`wget`，就不需要任何参数
{% highlight bash %}
wget https://octodex.github.com/images/welcometocat.png
{% endhighlight %}

##### Conclusion:
简单文件下载，且不需要安装额外资源，可以使用curl with `-O`，如果有批量下载任务，可以考虑wget。测试Web app 或者Web API， curl会更合适。

##### Others:
最后附上一些附加内容，更详细的`curl`的对照表（与其它的开源工具），另外一个是更友好更容易上手学习的curl的替代品。

- 查看对照表 [curl comparison table](http://curl.haxx.se/docs/comparison-table.html)
- 一个新”curl”工具– [httpie](https://github.com/jkbrzt/httpie)

