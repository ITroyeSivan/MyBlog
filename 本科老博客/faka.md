---
title: faka
author: Troy3e
avatar: 'https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/QQ%E5%9B%BE%E7%89%8720210427144151.jpg'
authorAbout: SteamID：888007034
authorDesc: Blizzard：TroyeSivan#51769
categories: 技术
comments: true
date: 2021-06-21 22:40:56
authorLink:
tags:
keywords:
description:
photos: https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/thumb-1920-1151994.png
---
最近正好在学习代码审计，但是因为期末的原因，一些大的cms都没有耐心仔细钻研。今天偶然在安全客上看到个很久之前做过的发卡系统，当时因为没有仔细研究导致印象不深，今天就以审计的角度再来看看这道题。
链接：https://www.anquanke.com/post/id/243357
https://blog.csdn.net/rfrder/article/details/115067196

## 越权

随便访问manage下的一个路由 /manage/Goods，都会跳转到登陆界面。看一下访问admin节点所需要的权限：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210621203800.png)

再看下system/node这个表，is_auth代表需要权限验证，is_login代表需要登录，那么看is_auth为0的地方就有可能存在越权。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210621204256.png)

漏洞点：application/admin/controller/Index.php

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210621213017.png)

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210621213052.png)

跟进下面的post处理，把post参数给$data，然后把password给md5,再对$data调用_form_filter这个回调方法：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210621213313.png)

_form里面那个array_merge将所有post请求存入数组，下面这_form_filter又用authorize判定权限，再结合我们一开始分析的数据库，这里只要post个authorize=3即可：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210621213740.png)

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210621214049.png)

成功创建高权限账号。
但是有一点没搞明白，就是：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210621214159.png)

不知道这里咋调试的，我在本地搭了环境一直连不上数据库不知道甚么原因。
还有就是越权根本原因，按下图所示，admin下的节点都是被控制的，那么为什么还会被绕过呢？

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210621214501.png)

这就要看判断权限的代码：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210621214716.png)

这里有默认值，node无论如何不可能为admin，就算访问/也是index/index/index。

## 任意文件读取
这个比较简单，seay都扫出来了：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210621214951.png)

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210621215040.png)

## 文件上传

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210621220215.png)

后面调着调着忘记写了，看开始的两个链接就行。这个地方也不是很难。最后那里控制后缀名还给我一种故意放了个漏洞的感觉，不知道开发者为啥要这么写：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210621223125.png)
强制拼接它不好么

## phar反序列化
phar反序列化给我的感觉是知道但不了解，随着学习的深入，我必须对这些基本的知识去重新挖掘一下。

参考：https://www.anquanke.com/post/id/243357#h2-3

## 总结
最近一个月脑子非常乱，一度想放弃安全转开发，但是最近一星期发生的许多事情让我坚定了在安全这条路上走下去的决心。
暂且定一下暑期的任务：
1、实习两个月。
2、每天都必须学一些新的知识并深入探索其原理，CTF题目或是复现漏洞。
3、冲一冲src。
4、有空学一学java。






