RMI全称是Remote Method Invocation，远程方法调用。  

# 基础

## 概述

### 简介

RMI 是 Java 提供的一个完善的简单易用的远程方法调用框架，采用客户/服务器通信方式，在服务器上部署了提供各种服务的远程对象，客户端请求访问服务器上远程对象的方法，它要求客户端与服务器端都是 Java 程序。

RMI 框架采用代理来负责客户与远程对象之间通过 Socket 进行通信的细节。RMI 框架为远程对象分别生成了客户端代理和服务器端代理。位于客户端的代理必被称为存根（Stub），位于服务器端的代理类被称为骨架（Skeleton）。
### 原理

当客户端调用远程对象的一个方法时，实际上是调用本地存根对象的相应方法。存根对象与远程对象具有同样的接口。存根采用一种与平台无关的编码方式，把方法的参数编码为字节序列，这个编码过程被称为参数编组。RMI 主要采用Java 序列化机制进行参数编组。

存根把以下请求信息发送给服务器：

- 被访问的远程对象的名字
- 被调用的方法的描述
- 编组后的参数的字节序列

服务器端接收到客户端的请求信息，然后由相应的骨架对象来处理这一请求信息，骨架对象执行以下操作：

- 反编组参数，即把参数的字节序列反编码为参数
- 定位要访问的远程对象
- 调用远程对象的相应方法
- 获取方法调用产生的返回值或者异常，然后对它进行编组
- 把编组后的返回值或者异常发送给客户

客户端的存根接收到服务器发送过来的编组后的返回值或者异常，再对它进行反编组，就得到调用远程方法的返回结果。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231030163423.png)

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231030163447.png)

方法调用从客户对象-经-存根（stub）、远程引用层（Remote Reference Layer）和传输层（Transport Layer）向下，传递给主机，然后再次经传输层，向上穿过远程调用层和骨干网（Skeleton），到达服务器对象。

- 存根：扮演着远程服务器对象的代理的角色，使该对象可被客户激活。
- 远程调用层：处理语义、管理单一或多重对象的通信，决定调用是应发往一个服务器还是多个。
- 传输层：管理实际的连接，并且追踪可以接受方法调用的远程对象。
- 骨干网：完成对服务器对象实际的方法调用，并获取返回值。返回值向下经远程引用层、服务器端的传输层传递回客户端，再向上经传输层和远程调用层返回。最后，存根获得返回值。

### 组成

RMI由3个部分构成：
第一个是rmiregistry（JDK提供的一个可以独立运行的程序，在bin目录下）
第二个是server端的程序，对外提供远程对象
第三个是client端的程序，想要调用远程对象的方法。

首先，先启动rmiregistry服务，启动时可以指定服务监听的端口，也可以使用默认的端口（1099）。
其次，server端在本地先实例化一个提供服务的实现类，然后通过RMI提供的Naming/Context/Registry（下面实例用的Registry）等类的bind或rebind方法将刚才实例化好的实现类注册到rmiregistry上并对外暴露一个名称。
最后，client端通过本地的接口和一个已知的名称（即rmiregistry暴露出的名称）再使用RMI提供的Naming/Context/Registry等类的lookup方法从RMIService那拿到实现类。这样虽然本地没有这个类的实现类，但所有的方法都在接口里了，便可以实现远程调用对象的方法了。

### 数据传递

Java程序中引用类型（不包括基本类型）的参数传递是按引用传递的，对于在同一个虚拟机中的传递时是没有问题的，因为的参数的引用对应的是同一个内存空间，在分布式系统中，由于对象不存在于同一个内存空间，虚拟机A的对象引用对于虚拟机B没有任何意义，那么怎么解决这个问题呢？

第一种：

将引用传递更改为值传递，也就是将对象序列化为字节，然后使用该字节的副本在客户端和服务器之间传递，而且一个虚拟机中对该值的修改不会影响到其他主机中的数据；

但是对象的序列化也有一个问题，就是对象的嵌套引用就会造成序列化的嵌套，这必然会导致数据量的激增，因此我们需要有选择进行序列化。

在Java中一个对象如果能够被序列化，需要满足下面两个条件之一：

- 是Java的基本类型；
- 实现java.io.Serializable接口（String类即实现了该接口）；

对于容器类，如果其中的对象是可以序列化的，那么该容器也是可以序列化的；
可序列化的子类也是可以序列化的；

第二种：

使用引用传递，每当远程主机调用本地主机方法时，该调用还要通过本地主机查询该引用对应的对象，在任何一台机器上的改变都会影响原始主机上的数据，因为这个对象是共享的；

RMI中的参数传递和结果返回可以使用的三种机制（取决于数据类型）：

- 简单类型：按值传递，直接传递数据拷贝；
- 远程对象引用（实现了Remote接口）：以远程对象的引用传递；
- 远程对象引用（未实现Remote接口）：按值传递，通过序列化对象传递副本，本身不允许序列化的对象不允许传递给远程方法；

## 示例

### 创建接口

创建一个接口Hello，该接口需要继承Remote接口，接口所定义的方法需要抛出RemoteException异常：

```java
package rmi;

import java.rmi.Remote;
import java.rmi.RemoteException;

public interface Hello extends Remote {
    public String Welcome(String name) throws RemoteException;
}
```

### 实现接口类

```java
package rmi.impl;

import rmi.Hello;

import java.rmi.RemoteException;
import java.rmi.server.UnicastRemoteObject;

/**
 * 远程接口实现类，必须继承UnicastRemoteObject
 * （继承RemoteServer->继承RemoteObject->实现Remote，Serializable），
 *  只有继承UnicastRemoteObject类，才表明其可以作为远程对象，被注册到注册表中供客户端远程调用
 * （补充：客户端lookup找到的对象，只是该远程对象的Stub（存根对象），
 *  而服务端的对象有一个对应的骨架Skeleton（用于接收客户端stub的请求，以及调用真实的对象）对应，
 *  Stub是远程对象的客户端代理，Skeleton是远程对象的服务端代理，
 *  他们之间协作完成客户端与服务器之间的方法调用时的通信。）
 */
public class HelloImpl extends UnicastRemoteObject implements Hello {
    //因为UnicastRemoteObject的构造方法抛出了RemoteException异常，
    //因此这里默认的构造方法必须写，也必须声明抛出RemoteException异常
    public HelloImpl() throws RemoteException {
    }

    @Override
    public String Welcome(String name) throws RemoteException {
        return "Hello " + name;
    }
}

```

如果一个远程类已经继承了其他类，无法再继承 UnicastRemoteObiect 类，那么可以在构造方法中调用 UnicastRemoteObject 类的静态 expotObject 方法，同样，远程类的构造方法也必须声明抛出 RemoteException

```java
package rmi.impl;

import rmi.Hello;

import java.rmi.RemoteException;
import java.rmi.server.UnicastRemoteObject;

public class HelloImpl2 implements Hello {
    @Override
    public String Welcome(String name) throws RemoteException {
        return "Hello " + name;
    }

    public HelloImpl2() throws RemoteException{
        //参数 port 指定监听的端口，如果取值为0，就表示监听任意一个匿名端口
        UnicastRemoteObject.exportObject(this, 0);
    }
}
```

### 创建服务端

```java
package rmi;

import rmi.impl.HelloImpl;

import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

public class Server {
    public static void main(String[] args) throws RemoteException {
        //创建对象
        Hello hello = new HelloImpl();
        // 本地主机上的远程对象注册表Registry的实例，
        // 并指定端口，这一步必不可少（Java默认端口是1099）
        Registry registry = LocateRegistry.createRegistry(1099);
        //绑定对象到注册表，并给它取名为hello
        registry.rebind("hello", hello);
    }
}
```

向注册器注册远程对象有三种方式：

```java
//创建远程对象
HelloService service1 = new HelloServiceImpl("service1");

//方式1:调用 java.i.registry.Registy 接口的 bind 或 rebind 方法
Registry registry = LocateRegistry.createRegistry(1099);
registry.rebind("HelloService1", service1);

//方式2:调用命名服务类 java.rmi.Naming 的 bind 或 rebind 方法
Naming.rebind("HelloService1"， service1);

//方式3:调用 JNDI API 的 javax.naming.Context 接口的 bind 或rebind 方法
Context namingContext = new InitialContext();
namingContext.rebind("rmi:HelloService1", service1);
```

### 客户端调用远程对象

```java
package rmi;

import java.rmi.NotBoundException;
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

public class Client {
    public static void main(String[] args) throws RemoteException, NotBoundException {
        //获取到注册表的代理
        Registry registry = LocateRegistry.getRegistry("localhost", 1099);
        //利用注册表的代理去查询远程注册表中名为hello的对象
        Hello hello = (Hello) registry.lookup("hello");
        //调用远程方法
        System.out.println(hello.Welcome("tom"));
    }
}
```

### 运行结果

![image-20231031111950484](C:\Users\86158\AppData\Roaming\Typora\typora-user-images\image-20231031111950484.png)

# 正文

## RMI通信过程

我们直接从⼀个例⼦开始演示RMI的流程吧。  

首先编写一个RMI Server：

```java
package rmi;

import java.rmi.Naming;
import java.rmi.Remote;
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;
import java.rmi.server.UnicastRemoteObject;
public class RMIServer {
    public interface IRemoteHelloWorld extends Remote {
        public String hello() throws RemoteException;
    }
    public class RemoteHelloWorld extends UnicastRemoteObject implements IRemoteHelloWorld {
        protected RemoteHelloWorld() throws RemoteException {
            super();
        }
        public String hello() throws RemoteException {
            System.out.println("call from");
            return "Hello world";
        }
    }
    private void start() throws Exception {
        RemoteHelloWorld h = new RemoteHelloWorld();
        LocateRegistry.createRegistry(1099);
        Naming.rebind("rmi://127.0.0.1:1099/Hello", h);
    }
    public static void main(String[] args) throws Exception {
        new RMIServer().start();
    }
}
```

⼀个RMI Server分为三部分：  

1. ⼀个继承了 java.rmi.Remote 的接⼝，其中定义我们要远程调⽤的函数，比如这⾥的 hello()
2. 一个实现了此接口的类
3. ⼀个主类，⽤来创建Registry，并将上面的类实例化后绑定到⼀个地址。这就是我们所谓的Server
   了。  

接着我们编写⼀个RMI Client：  

```java
package rmi;

import java.rmi.Naming;

public class RMIClient {
    public static void main(String[] args) throws Exception {
        RMIServer.IRemoteHelloWorld hello = (RMIServer.IRemoteHelloWorld)Naming.lookup("rmi://192.168.0.120:1099/Hello");
        String ret = hello.hello();
        System.out.println(ret);
    }
}
```

捋⼀捋这整个过程，⾸先客户端连接Registry，并在其中寻找Name是Hello的对象，这个对应数据流中的Call消息；然后Registry返回⼀个序列化的数据，这个就是找到的Name=Hello的对象，这个对应数据流中的ReturnData消息；客户端反序列化该对象，发现该对象是⼀个远程对象，地址在 192.168.135.142:33769 ，于是再与这个地址建立TCP连接；在这个新的连接中，才执行真正远程方法调用，也就是 hello() 。  

借用下图来说明这些元素间的关系：  

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231031133605.png)

RMI Registry就像⼀个网关，他⾃⼰是不会执⾏远程方法的，但RMI Server可以在上⾯注册⼀个Name
到对象的绑定关系； RMI Client通过Name向RMI Registry查询，得到这个绑定关系，然后再连接RMI
Server；最后，远程⽅法实际上在RMI Server上调⽤。  

## 攻击RMI Registry

总结一下，一个RMI过程有以下三个参与者：  

- RMI Registry
- RMI Server
- RMI Client  

但是为什么示例代码只有两个部分呢？原因是，通常我们在新建一个RMI Registry的时候，都会直接绑定一个对象在上面，也就是说我们示例代码中的Server其实包含了Registry和Server两部分：  

```java
LocateRegistry.createRegistry(1099);
Naming.bind("rmi://127.0.0.1:1099/Hello", new RemoteHelloWorld());
```

第一行创建并运行RMI Registry，第二行将RemoteHelloWorld对象绑定到Hello这个名字上。  

Naming.bind 的第一个参数是一个URL，形如： rmi://host:port/name 。其中，host和port就是RMI Registry的地址和端口，name是远程对象的名字。

如果RMI Registry在本地运行，那么host和port是可以省略的，此时host默认是 localhost ，port默认
是 1099 ：  

```java
Naming.bind("Hello", new RemoteHelloWorld());
```

RMI会给我们带来哪些安全问题呢？  

1. 如果我们能访问RMI Registry服务，如何对其攻击？
2. 如果我们控制了目标RMI客户端中 Naming.lookup 的第一个参数（也就是RMI Registry的地
   址），能不能进行攻击？

### 1.客户端攻击RMI Registry

只能客户端打注册中心。注册中心的交互主要是这一句话：

```java
Naming.bind("rmi://127.0.0.1:1099/Hello", h);
```

这里的交互方式不只是只有 bind，还有其他的一系列方式：

- list
- bind
- rebind
- unbind
- lookup

这几种方法位于 `RegistryImpl_Skel#dispatch` 中，如果存在对传入的对象调用 `readObject()` 方法，则可以利用，`dispatch` 里面对应关系如下：

- 0 —– bind
- 1 —– list
- 2 —– lookup
- 3 —– rebind
- 4 —– unbind

除了 list 和 lookup 两个，其余的交互在 8u121 之后都是需要 localhost 的。

#### 使用list()方法进行攻击

用 `list()` 方法可以列出目标上所有绑定的对象：

```java
package rmi;

import java.rmi.Naming;

public class RMIClient {
    public static void main(String[] args) throws Exception {
        String[] s = Naming.list("rmi://127.0.0.1:1099");
        System.out.println(s);
    }
}
```

输出：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231101174650.png)

后续攻击方法需要反序列化知识，所以这块暂时搁置，以后再补。



# 参考链接

1、https://github.com/phith0n/JavaThings

2、https://blog.csdn.net/cold___play/article/details/132086492

3、https://drun1baby.top/2022/07/23/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B9%8BRMI%E4%B8%93%E9%A2%9802-RMI%E7%9A%84%E5%87%A0%E7%A7%8D%E6%94%BB%E5%87%BB%E6%96%B9%E5%BC%8F

3、https://www.anquanke.com/post/id/204740#h3-10
