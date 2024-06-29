---
title: xss-跨站脚本攻击1
author: Troye
avatar: https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20160708223941_YQncj.jpeg
authorLink: 
authorAbout: 
authorDesc: 
categories: 技术
comments: true
date: 2020-02-11 00:41:25
tags:
keywords:
description:
photos: https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/源氏Genji守望先锋4k壁纸_彼岸图网.jpg
---
xss（Cross Site Scripting），全称跨站脚本攻击。XSS攻击通常指的是通过利用网页开发时留下的漏洞，通过巧妙的方法注入恶意指令代码到网页，使用户加载并执行攻击者恶意制造的网页程序。这些恶意网页程序通常是JavaScript，但实际上也可以包括Java、 VBScript、ActiveX、 Flash 或者甚至是普通的HTML。攻击成功后，攻击者可能得到包括但不限于更高的权限（如执行一些操作）、私密网页内容、会话和cookie等各种内容。
xss靶场记录：
level1：点击图片开始挑战。查看回报发现name=什么，就在欢迎用户的后面输出什么，由此可以在name=后面构造：<script>alert(‘xss’)</script>。效果如下：
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200211001234.png)
level2：先分析源码，发现在框中输入什么，就会在value=“”>中输出什么，按照第一关的方法输入失败了，查看源码可知需要构造闭合：
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200211001342.png)
构造闭合：
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200211001438.png)
前面加上”>来完成value=“”>这个语句。后面加上//起注释的作用，这样javascript引擎就会执行<script>alert(1)</script>这句语句。
level3：首先查看源码发现和level2差不多，于是首先套用level2的方法尝试，但是<和>被过滤了：
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200211001536.png)
尝试构造onclick事件触发xss：
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200211001618.png)
注：这个地方还不是很懂，只能先记住了。
level4：类型和上面一样，首先尝试"><script>alert(1)</script>//，
但是肯定是不行的，被过滤了。其次尝试构造onclick事件触发xss，成功。
level5：首先尝试"><script>alert(1)</script>//，发现变成了"><scr_ipt>alert(1)</script>//。其次尝试构造onclick事件，发现变成了" o_nclick='alert(1)'。利用没有过滤尖括号，构造a标签再尝试利用a标签的href属性执行javascript:伪协议。
"><a href='javascript:alert(1)',没有对javascript进行过滤，触发xss。
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200211001800.png)
level6：应该有很多种方法，我找到了两种。一种是最简单的"><script>alert(1)</script>//，把第一个script的s改为大写即"><Script>alert(1)</script>//。第二种是将上一关中href的h改为大写即可。
注：其实还有几种，只是变化不大就不一一列举了。
level7：首先尝试了script（即那个最简单的，这里简称script）和onclick，发现script是直接消失，而onclick则是少了on，由此可知应该不能利用大小写来绕过。然后尝试用重复写来绕过，其中scriptscript（这个有点蠢）和ononclick都失败了，接着又试了各种href的，都失败了，各种玄学问题。最后找出来一个：
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200211001904.png)
将中间的script扣掉之后两边连起来还是script，我称之为小天才行为。

注：明日开始爆肝学习QAQ

