---
title: Java反序列化学习
author: Troy3e
avatar: >-
  https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/QQ%E5%9B%BE%E7%89%8720210308190505.jpg
authorAbout: SteamID：888007034
authorDesc: Blizzard：TroyeSivan#51769
categories: 技术
comments: true
date: 2021-03-25 16:03:39
authorLink:
tags:
keywords:
description:
photos: https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/thumb-1920-1138790.jpg
---
这几天做了很多题 经验总结一句话就是：要多分析源码。
Java一直没去深入研究 比赛的时候自己根本没法做 还是连续两场都碰上 非常后悔 从今往后分析平台源码也得加入每日学习内容了。
进入正文
——————————————————————————————————
## Java反序列化有啥用?
1)把对象的字节序列永久地保存到硬盘上，通常存放在一个文件中；
2)在网络上传送对象的字节序列
## Java中的API实现
 Java.io.ObjectOutputStream&&Java.io.ObjectInputStream
 ObjectOutputStream类 --> writeObject()
该方法对参数指定的obj对象进行序列化，把字节序列写到一个目标输出流中，按Java的标准约定是给文件一个.ser扩展名。
想起来前几天补考Java才刚学过hhh
打开eclipse试一下：

    package example;
    
    import java.io.*;
    
    public class example3_1{
	    public static void main(String args[]) throws Exception{
		     String obj = "emt";
	            // 将序列化对象写入文件object.txt中
	            FileOutputStream fos = new FileOutputStream("troy3e.ser");
	            ObjectOutputStream os = new ObjectOutputStream(fos);
	            os.writeObject(obj);
	            os.close();
    
	            // 从文件object.txt中读取数据
	            FileInputStream fis = new FileInputStream("troy3e.ser");
	            ObjectInputStream ois = new ObjectInputStream(fis);
    
	            // 通过反序列化恢复对象obj
	            String obj2 = (String)ois.readObject();
	            System.out.println(obj2);
	            ois.close();
    	}
    }

先通过输入流创建一个文件，再调用ObjectOutputStream类的 writeObject方法把序列化的数据写入该文件;然后调用ObjectInputStream类的readObject方法反序列化数据并打印数据内容。
对于类的话 实现Serializable和Externalizable接口的类的对象才能被序列化。（class Troy3e implements Serializable）
## 漏洞产生原因
与PHP反序列化是一个道理，如果Java应用对用户输入，即不可信数据做了反序列化处理，那么攻击者可以通过构造恶意输入，让反序列化产生非预期的对象，非预期的对象在产生过程中就有可能带来任意代码执行。
## 实战分析
一、[V&N2020 公开赛]EasySpringMVC
WP本地记录。
二、NepCTF
WP本地记录。

持续更新。
PS：WP以后会做一个合集，不再单发水博客。