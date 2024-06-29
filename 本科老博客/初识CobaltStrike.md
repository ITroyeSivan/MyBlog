---
title: 初识CobaltStrike
author: Troy3e
avatar: 'https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/QQ%E5%9B%BE%E7%89%8720210427144151.jpg'
authorAbout: SteamID：888007034
authorDesc: Blizzard：TroyeSivan#51769
categories: 技术
comments: true
date: 2021-05-23 23:22:12
authorLink:
tags:
keywords:
description:
photos: https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/thumb-1920-1147728.png
---
好久没更新博客了，这几星期感觉做了很多事情，又感觉什么事都没干。又想挖洞，又想往开发方向发展，比赛也零零碎碎打了不少。
马上就要被拉去做hvv免费苦力了，在这之前学习一下CS神器的使用。

## CobaltStrike是什么
CobaltStrike是一款渗透测试神器，被业界人称为CS神器。CobaliSitike分为客户端与服务端，服务端是一个，客户端可以有多个，可被团队进行分布式协团操作。
Cobaltsitrike集成了端口转发、服务扫描，自动化溢出，多模式端口监听，windows exe木马生成，windows dl木马生成，java木马生成，office宏病毒生成，木马捆绑。钓鱼攻击包括:站点克隆，目标信息获取，java执行，浏览器自动攻击等等。

## CobaltStrike的安装
我采用的是最近刚放出来的4.3的破解版。

链接：https://pan.baidu.com/s/1Kzv0fRcYthoP2QYHq98vTw 
提取码：tvmy 
来源于某公众号，使用后造成的风险自行承担。
链接挂了的话。。。挂了就挂了，应该也没人看我博客。
一、服务端
团队服务器最好搭在linux服务器上。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210523234100.png)

没什么环境要求，jdk8即可。
启动命令：
./teamserver 你的ip 你的密码

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210523234508.png)
上图所示便是成功启动。
端口默认50050，云服务器记得手动去控制台开一下。
如果你这里报错的话，请检查：
1、java版本是否正确（请使用jdk8）
2、是否更改了teamserver里的文件，比如修改了密码。这里说明一下，文件里的密码123456是不能更改的。密码直接在命令里指定即可。

二、客户端
客户端我个人认为还是windows比较方便些，直接双击exe运行即可：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210523235042.png)

按你自己配置的信息写入。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210523235121.png)

## 目录信息及参数说明

CobaltStrike—些主要文件功能如下:

    agscript:扩展应用的脚本
    c2lint:用于检查profile的错误和异常
    teamserver:服务器端启动程序
    cobaltstrike.jar: CobaltStrike核心程序
    cobaltstrike.auth:用于客户端和服务器端认证的文件，客户端和服务端有一个一模一样的
    cobaltstrike.store:秘钥证书存放文件

—些目录作用如下:

    data:用于保存当前TeamServer的一些数据
    download:用于存放在目标机器下载的数据
    upload: 上传文件的目录
    logs:日志文件，包括Web日志、Beacon日志、截图日志、下载日志、键盘记录日志等
    third-party:第三方工具目录

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210523235500.png)

Cobalt Strike

    New Connection   # 新建连接，支持连接多个服务器端
    Preferences   # 设置Cobal Strike界面、控制台、以及输出报告样式、TeamServer连接记录
    Visualization   # 主要展示输出结果的视图
    VPN Interfaces   # 设置VPN接口
    Listenrs   # 创建监听器
    Script Manager   # 脚本管理，可以通过AggressorScripts脚本来加强自身，能够扩展菜单栏，Beacon命令行，提权脚本等
    Close   # 退出连接

View

    Applications   # 显示受害主机的应用信息
    Credentials   # 显示所有以获取的受害主机的凭证，如hashdump、Mimikatz
    Downloads   # 查看已下载文件
    Event Log   # 主机上线记录以及团队协作聊天记录
    Keystrokes   # 查看键盘记录结果
    Proxy Pivots   # 查看代理模块
    Screenshots   # 查看所有屏幕截图
    Script Console   # 加载第三方脚本以增强功能 
    Targets   # 显示所有受害主机
    Web Log    # 所有Web服务的日志

Attacks
Packages

    HTML Application   # 生成(executable/VBA/powershell)这三种原理实现的恶意HTA木马文件
    MS Office Macro   # 生成office宏病毒文件
    Payload Generator   # 生成各种语言版本的payload
    USB/CD AutoPlay   # 生成利用自动播放运行的木马文件
    Windows Dropper   # 捆绑器能够对任意的正常文件进行捆绑(免杀效果差)
    Windows Executable   # 生成可执行exe木马
    Windows Executable(Stageless)   # 生成无状态的可执行exe木马

Web Drive-by

    Manage   # 对开启的web服务进行管理
    Clone Site   # 克隆网站，可以记录受害者提交的数据
    Host File   # 提供文件下载，可以选择Mime类型
    Scripted Web Delivery   # 为payload提供web服务以便下载和执行，类似于Metasploit的web_delivery 
    Signed Applet Attack   # 使用java自签名的程序进行钓鱼攻击(该方法已过时)
    Smart Applet Attack   # 自动检测java版本并进行攻击，针对Java 1.6.0_45以下以及Java 1.7.0_21以下版本(该方法已过时)
    System Profiler   # 用来获取系统信息，如系统版本，Flash版本，浏览器版本等
    Spear Phish   # 鱼叉钓鱼邮件

Reporting

    Activity Report   # 活动报告
    Hosts Report   # 主机报告
    Indicators of Compromise   # IOC报告：包括C2配置文件的流量分析、域名、IP和上传文件的MD5 hashes
    Sessions Report   # 会话报告
    Social Engineering Report   # 社会工程报告：包括鱼叉钓鱼邮件及点击记录
    Tactics, Techniques, and Procedures   # 战术技术及相关程序报告：包括行动对应的每种战术的检测策略和缓解策略
    Reset Data   # 重置数据
    Export Data   # 导出数据，导出.tsv文件格式

Help

    Homepage   # 官方主页
    Support   # 技术支持
    Arsenal   # 开发者
    System information   # 版本信息
    About   # 关于

## CobaltStrike的使用
一、创建监听器Listener
点击Cobalt Strike选择listeners，点击下方的add

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210524132747.png)

    name：为监听器名字，可任意
    payload：payload类型
    HTTP Hosts: shell反弹的主机，也就是我们kali的ip
    HTTP Hosts(Stager): Stager的马请求下载payload的地址
    HTTP Port(C2): C2监听的端口

Beacon为内置的Listener，即在目标主机执行相应的payload，获取shell到CS上;其中包含DNS、HTTP、HTTPS、SMB。Beacon可以选择通过DNS还是HTTP协议出口网络，你甚至可以在使用Beacon通讯过程中切换HTTP和DNS。其支持多主机连接，部署好Beacon后提交一个要连回的域名或主机的列表，Beacon将通过这些主机轮询。目标网络的防护团队必须拦截所有的列表中的主机才可中断和其网络的通讯。通过种种方式获取she以后（(比如直接运行生成的exe)，就可以使用Beacon了。
Foreign为外部结合的Listener，常用于MSF的结合，例如获取meterpreter到MSF上。

二、创建攻击Attacks
这里Attacks有几种，如下：

· HTML Application 　　　　　　 生成一个恶意HTML Application木马，后缀格式为 .hta。通过HTML调用其他语 言的应用组件进行攻击，提供了 可执行文件、PowerShell、VBA三种方法。

· MS Office Macro 　　　　　　 生成office宏病毒文件；

· Payload Generator 　　　　　 生成各种语言版本的payload，可以生成基于C、C#、COM Scriptlet、Java、Perl、 PowerShell、Python、Ruby、VBA等的payload

· Windows Executable 　　　　　生成32位或64位的exe和基于服务的exe、DLL等后门程序

· Windows Executable(S)　　　　用于生成一个exe可执行文件，其中包含Beacon的完整payload，不需要阶段性的请求。与Windows Executable模块相比，该模块额外提供了代理设置，以便在较为苛刻的环境中进行渗透测试。该模块还支持powershell脚本，可用于将Stageless Payload注入内存

Windows Executable

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210524132901.png)

双击生成的程序，记得把windows自带的保护关掉

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210524133240.png)

## 对被控主机的操作

    Interact       打开beacon
    Access 
	    dump hashes   获取hash
	    Elevate       提权
    	Golden Ticket 生成黄金票据注入当前会话
	    MAke token    凭证转换
	    Run Mimikatz  运行 Mimikatz 
	    Spawn As      用其他用户生成Cobalt Strike的beacon
    Explore
    	Browser Pivot 劫持目标浏览器进程
	    Desktop(VNC)  桌面交互
	    File Browser  文件浏览器
	    Net View      命令Net View
	    Port scan     端口扫描
	    Process list  进程列表
	    Screenshot    截图
    Pivoting
    	SOCKS Server 代理服务
	    Listener     反向端口转发
	    Deploy VPN   部署VPN
    Spawn            新的通讯模式并生成会话
    Session          会话管理，删除，心跳时间，退出，备注


一、命令说明
beacon处输入help：

    Beacon Commands
    ===============
    Command                   Description
    -------                   -----------
    argue                     进程参数欺骗
    blockdlls                 在子进程中阻止非Microsoft的DLLs文件
    browserpivot              注入受害者浏览器进程
    bypassuac                 绕过UAC
    cancel                    取消正在进行的下载
    cd                        切换目录
    checkin                   强制让被控端回连一次
    clear                     清除beacon内部的任务队列
    connect                   通过TCP连接到Beacon
    covertvpn                 部署Covert VPN客户端
    cp                        复制文件
    dcsync                    从DC中提取密码哈希
    desktop                   远程VNC
    dllinject                 反射DLL注入进程
    dllload                   使用LoadLibrary将DLL加载到进程中
    download                  下载文件
    downloads                 列出正在进行的文件下载
    drives                    列出目标盘符
    elevate                   尝试提权
    execute                   在目标上执行程序(无输出)
    execute-assembly          在目标上内存中执行本地.NET程序
    exit                      退出beacon
    getprivs                  对当前令牌启用系统权限
    getsystem                 尝试获取SYSTEM权限
    getuid                    获取用户ID
    hashdump                  转储密码哈希值
    help                      帮助
    inject                    在特定进程中生成会话
    jobkill                   杀死一个后台任务
    jobs                      列出后台任务
    kerberos_ccache_use       从ccache文件中导入票据应用于此会话
    kerberos_ticket_purge     清除当前会话的票据
    kerberos_ticket_use       从ticket文件中导入票据应用于此会话
    keylogger                 键盘记录
    kill                      结束进程
    link                      通过命名管道连接到Beacon
    logonpasswords            使用mimikatz转储凭据和哈希值
    ls                        列出文件
    make_token                创建令牌以传递凭据
    mimikatz                  运行mimikatz
    mkdir                     创建一个目录
    mode dns                  使用DNS A作为通信通道(仅限DNS beacon)
    mode dns-txt              使用DNS TXT作为通信通道(仅限D beacon)
    mode dns6                 使用DNS AAAA作为通信通道(仅限DNS beacon)
    mode http                 使用HTTP作为通信通道
    mv                        移动文件
    net                       net命令
    note                      给当前目标机器备注       
    portscan                  进行端口扫描
    powerpick                 通过Unmanaged PowerShell执行命令
    powershell                通过powershell.exe执行命令
    powershell-import         导入powershell脚本
    ppid                      为生成的post-ex任务设置父PID
    ps                        显示进程列表
    psexec                    使用服务在主机上生成会话
    psexec_psh                使用PowerShell在主机上生成会话
    psinject                  在特定进程中执行PowerShell命令
    pth                       使用Mimikatz进行传递哈希
    pwd                       当前目录位置
    reg                       查询注册表
    rev2self                  恢复原始令牌
    rm                        删除文件或文件夹
    rportfwd                  端口转发
    run                       在目标上执行程序(返回输出)
    runas                     以另一个用户权限执行程序
    runasadmin                在高权限下执行程序
    runu                      在另一个PID下执行程序
    screenshot                屏幕截图
    setenv                    设置环境变量
    shell                     cmd执行命令
    shinject                  将shellcode注入进程
    shspawn                   生成进程并将shellcode注入其中
    sleep                     设置睡眠延迟时间
    socks                     启动SOCKS4代理
    socks stop                停止SOCKS4
    spawn                     生成一个会话 
    spawnas                   以其他用户身份生成会话
    spawnto                   将可执行程序注入进程
    spawnu                    在另一个PID下生成会话
    ssh                       使用ssh连接远程主机
    ssh-key                   使用密钥连接远程主机
    steal_token               从进程中窃取令牌
    timestomp                 将一个文件时间戳应用到另一个文件
    unlink                    断开与Beacon的连接
    upload                    上传文件
    wdigest                   使用mimikatz转储明文凭据
    winrm                     使用WinRM在主机上生成会话
    wmi                       使用WMI在主机上生成会话

二、提权
右键->Access->Elevate

我这里因为是win10，自带的两个payload都不能成功提权。

若提权成功，会返回来一个管理员的shell。
当然我这里还是返回了一个普通权限：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210524134546.png)

三、抓取hash和dump明文密码

右键->Access->DumpHashes
利用mimikatz抓取明文密码：右键->Access->Run Mimikatz

抓取密码哈希，也可以直接输入hashdump
使用mimikatz抓取明文密码，也可以直接输入logonpasswords

抓取完之后点击凭证信息，就会显示我们抓取过的哈希或明文。由于我没有管理员权限，所以并没有尝试。

四、利用被控主机建立Socks4代理

当我们控制的主机是一台位于公网和内网边界的服务器，我们想利用该主机继续对内网进行渗透，于是，我们可以利用CS建立socks4A代理

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210524135243.png)

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210524135329.png)

然后vim /etc/proxychains.conf ，在文件末尾添加socks4代理服务器

socks4 服务端ip 9071
使用proxychains代理扫描内网主机
proxychains nmap -sP x.x.x.x/24

我们还可以通过隧道将整个msf带进目标内网
点击View->Proxy Pivots，选择Socks4a Proxy,点击Tunnel:

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210524135810.png)

打开msf对内网进行扫描

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210524135924.png)

关闭socks
beacon>socks stop

五、注入进程、键盘监控
右键被控主机->Explore->Process List

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210524140801.png)

即可列出进程列表

kill为杀死该进程，refresh为刷新该进程，inject则是把beacon注入进程，log keystrokes为键盘记录，screenshot为截图，stea token为窃取运行指定程序的用户令牌

1、Inject注入进程
选择进程，点击Inject，随后选择监听器，点击choose，即可发现CobaltStrike弹回了目标机的一个新会话，这个会话就是成功注入到某进程的beacon会话。该功能可以把你的beacon会话注入到另外—个程序之中，注入之后，除非那个正常进程被杀死了，否则我们就一直可以控制该主机了。

inject  进程PID  进程位数  监听

单击一个进程，单击inject，选择监听器，就会弹回来beacon。

2、键盘记录
任意选择一个进程，点击log keystrokes，即可监听该主机的键盘记录

keylogger 进程PID 进程位数

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210524141629.png)

(硬核打码.jpg

查看键盘记录结果需要点击上面那个钥匙按钮。注意这里是监听所有的键盘记录，而不只是选中进程的键盘记录。

六、浏览器代理Browser Pivot
使用浏览器代理功能，我们可以注入到目标机器的浏览器进程中。然后在本地浏览器中设置该代理，这样，我们可以在本地浏览器上访问目标机器浏览器已经登录的网站，而不需要登录。但是目前浏览器代理只支持IE浏览器。如下，目标主机的IE浏览器目前在访问fofa，并且已登录。现在我们想利用浏览器代理在本地利用目标主机身份进行访问。

explore->Browser Pivot
随便选一个进程，点击开始。
![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210524142214.png)

然后视图代理信息可以看到刚刚建立的浏览器代理。这里的意思是，TeamServer监听59398和26193端口。流程是这样，我们将流量给59398端口，59398端口将流量给26193端口，26193将流星给目标主机的26193端口。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210524142546.png)

这里懒得试了，放个图。

七、生成黄金票据注入当前会话（Golden Ticket）
暂略。

八、凭证转换
暂略。

九、端口扫描

    portscan 192.168.10.1-192.168.10.10 22,445 arp  1024
    portscan 192.168.10.1-192.168.10.10 22,445 icmp 1024
    portscan 192.168.10.1-192.168.10.10 22,445 none 1024
 
    一般我们直接运行命令
    portscan 192.168.1.0/24 22,445,1433,3306

十、哈希传递攻击或SSH远程登录
暂略。

十一、还有很多，暂时不写了

## 小结
还没写完，有空继续学。