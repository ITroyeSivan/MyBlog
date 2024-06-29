---
title: upload-labs
author: Troye.
avatar: https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20160708223941_YQncj.jpeg
authorLink: hojun.cn
authorAbout: 
authorDesc: 
categories: 技术
comments: true
date: 2020-02-19 18:19:24
tags:
keywords:
description:
photos: https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/929254.png
---
Uploads-labs
Pass-01：bp抓包将jpg后缀改成php或者使用noscript插件禁用js即可。
Pass-02：首先查看源码，发现本关好像只判断content type，所以bp抓包修改即可。
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200219182453.png)
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200219182501.png)
Pass-03：查看源码可以看到过滤了asp、aspx、php、jsp。
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200219182637.png)
但是后面的有点看不明白。如果只看前面过滤的这四个那么可以尝试用php1、php2这种来绕过。
幸好这里这种方法是可行的，不然gg：
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200219182712.png)
Pass-04：看了下网上的做法，这里几乎过滤了所有的后缀，除了htaccess，所以首先上传一个.htaccess文件包含下面的内容：<FilesMatch "11.jpg">
SetHandler application/x-httpd-php
</FilesMatch>。
这句话可以让上传的所有文件都解析成php（11.jpg是指下面要传的文件）
直接bp抓包修改：
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200219182747.png)
接着把11.jpg这个文件提交：
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200219182811.png)
然后直接访问即可
Pass-05：这一关加上了对htaccess的过滤，但是没有限制大小写，所以尝试用大小写绕过：
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200219182833.png)
可能是服务器出了点问题，上传的文件都不能成功执行，但是问题不大，只要学方法就行了。
Pass-06：这一关少了首位去空这一限制，随便试了个. php就成功了。
Pass-07：这一关少了去除文件名末尾的点，那就试试在最后加个点。
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200219183018.png)
成功。
Pass-08：这一关少了去除字符串。但是不太明白是啥意思，一开始以为要转码进行绕过，看了网上的才知道是咋回事。这里没有对后缀名进行去::$DATA处理，利用windows特性，可在后缀名中加” ::$DATA”绕过：
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200219183103.png)
Pass-09：查看源码，发现之前几关出现的限制全都有。但这里可以利用首尾去空和删除文件末尾的点来构造：test.php. .,这样在过滤之后去掉了最后的点和空格，剩下了test.php.，就能绕过了。
Pass-10：源码中有这样一句话
$file_name = str_ireplace($deny_ext,"", $file_name);
网上看了下知道这是将后缀替换成空的，所以和xss靶场一样重复写就行了：pphphp
Pass-11：这一关是白名单，只允许上传gif、png、jpg。源码还有这一行：
$img_path=$_GET['save_path']."/".rand(10,99).date("YmdHis").".".$file_ext;
Get方式上传一个变量save_path和随机日期来组成文件上传地址，这样就实现了对文件名的控制。但是我们可以利用00截断：
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200219183210.png)
Pass-12：这一关就是将上一关的get改成了post，一样是00绕过，但是这里需要在二进制里修改，因为post不像get可以自动解码
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200219183251.png)
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200219183323.png)
Pass-13：开始模仿大佬操作，首先学习制作一个图片马。制作好以后上传即可。由于本关根据前两个字节判断文件类型，所以随便构造一个就行。
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200219183419.png)
Pass-14：本关的重点是这个getimagesize（）函数
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200219183456.png)
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200219183505.png)
绕过方法和上一关相同。
Pass-15：这里用到php_exif模块来判断文件类型，还是直接就可以利用图片马就可进行绕过。
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200219183559.png)
注：exif_imagetype() 读取一个图像的第一个字节并检查其签名。
Pass-16：此关首先判断文件的后缀名、content-type，然后用imagecreatefromjpeg函数判断函数是否是jpg函数（还有png和gif就略去不说了），如果是的话对图片进行二次渲染。二次渲染可能会导致我们之前插入的php语句被删去，这里只要找到二次渲染后留下的那部分把php语句插在那里就行了。别的也没什么好说的了，主要是已经不太懂了。
![avatar](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200219183633.png)
Pass-17：这里先将文件上传到服务器，然后通过rename修改名称，再通过unlink删除文件，因此可以通过条件竞争的方式在unlink之前，访问webshell。方法就是用bp不断发送并不断访问，在大量的尝试中可能会有几次是访问成功的。
Pass-18：本关对文件后缀名做了白名单判断，然后会一步一步检查文件大小、文件是否存在等等，将文件上传后，对文件重新命名，同样存在条件竞争的漏洞。可以不断利用burp发送上传图片马的数据包，由于条件竞争，程序会出现来不及rename的问题，从而上传成功。
Pass-19：看了网上的很多帖子都基本上不相同，就不尝试了，超出自身能力了。
