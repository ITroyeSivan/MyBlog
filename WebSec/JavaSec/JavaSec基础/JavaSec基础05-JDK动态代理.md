好像用处不大，但是反序列化会用到，就简单学习一下。

# Java的代理模式

定义：为其他对象提供一种代理以控制对这个对象的访问。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231108142743.png)

## 1、静态代理

以租房为例，我们一般用租房软件、找中介或者找房东。这里的中介就是代理者。

`Rent.java`：这是一个接口，可以抽象的理解为房源，作为房源，它有一个方法`rent()`为**租房**

```java
public interface Rent {
    //租房的接口
    void rent();
}
```

`Host.java`：这个类就是房东，作为房东，他需要实现`Rent.java`这一个接口，并且要实现接口的`rent()`方法。

```java
public class Host implements Rent {
    @Override
    public void rent() {
        System.out.println("出租房子");
    }
}
```

`Proxy.java`：这是一个类，这个类是中介，也就是代理，他需要有房东的房源，然而我们通常不会继承房东，而会将房东作为一个私有的属性`host`，我们通过`host.rent()`来实现租房的方法。

```java
public class Proxy {

    private Host host;

    public Proxy(){
        
    }
    
    public Proxy(Host host){
        this.host = host;
    }

    @Override
    public void rent() {
        System.out.println("收中介费");
        System.out.println("中介带你看房");
        host.rent();
        System.out.println("签租赁合同");
    }
}
```

`Client.java`租客去找中介看房

```java
public class Client {

    public static void main(String[] args){
        //定义租房
        Host host = new Host();
        //定义中介
        Proxy proxy = new Proxy(host);
        //中介租房
        proxy.rent();
    }
}
```

返回信息：

```
交中介费
租了一间房子。。。
中介负责维修管理
```

这就是静态代理，因为中介这个代理类已经事先写好了，只负责代理租房业务。

**优点：**

- 可以使得我们的真实角色更加纯粹 . 不再去关注一些公共的事情 .
- 公共的业务由代理来完成 . 实现了业务的分工 ,
- 公共业务发生扩展时变得更加集中和方便 .

**缺点 :**

- 一个真实类对应一个代理角色，代码量翻倍，开发效率降低 .

中介的例子可能不好理解，这里再以实际生产中的业务CRUD为例。

定义一个接口UserService.java:

```java
package jdkproxy;

public interface UserService {
    public void add();
    public void delete();
    public void update();
    public void query();
}

```

定义一个真实对象完成这些操作：

```java
package jdkproxy;

public class UserServiceImpl implements UserService{

    @Override
    public void add() {
        System.out.println("增加了一个用户");
    }

    @Override
    public void delete() {
        System.out.println("删除了一个用户");
    }

    @Override
    public void update() {
        System.out.println("修改了一个用户");
    }

    @Override
    public void query() {
        System.out.println("查询了一个用户");
    }
}

```

现在需要实现一个日志功能，该如何实现呢？

在实现类上增加代码比较麻烦。若使用代理来做，能够在 不改变原来的业务情况下，实现此功能。

增加一个代理类来处理日志：

```java
package jdkproxy;

public class UserServiceProxy implements UserService{
    private UserServiceImpl userService;

    public void setUserService(UserServiceImpl userService) {
        this.userService = userService;
    }

    @Override
    public void add() {
        log("add");
        userService.add();
    }

    @Override
    public void delete() {
        log("delete");
        userService.delete();
    }

    @Override
    public void update() {
        log("update");
        userService.update();
    }

    @Override
    public void query() {
        log("query");
        userService.query();
    }

    public void log(String msg){
        System.out.println("[Debug]使用了 " + msg +"方法");
    }
}
```

修改Client类：

```java
package jdkproxy;

public class Client {
    public static void main(String[] args) {
        UserServiceImpl userService = new UserServiceImpl();
        UserServiceProxy proxy = new UserServiceProxy();
        proxy.setUserService(userService);
        proxy.add();
    }
}
```

输出：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231108161236.png)

## 2、动态代理

在上面的静态代理中，每多一个房东，就需要多一个中介。在开发中，如果有多个“中介”，那代码量就会过大。

动态代理的出现就是为了解决上面静态代理的缺点。

- 动态代理的角色和静态代理的一样。需要一个实体类，一个代理类，一个启动器。
- 动态代理的代理类是动态生成的，静态代理的代理类是我们提前写好的。

动态代理的实现类：

```java
package jdkproxy;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

public class DynamicProxy implements InvocationHandler {

    //被代理的接口
    private UserService userService;

    public void setUserService(UserService userService){
        this.userService = userService;
    }

    //动态生成代理类实例
    public Object getProxy(){
        Object obj = Proxy.newProxyInstance(this.getClass().getClassLoader(), userService.getClass().getInterfaces(), this);
        return obj;
    }

    // 处理代理类实例，并返回结果
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        log(method);
        Object obj = method.invoke(userService, args);
        return obj;
    }

    //业务自定义需求
    public void log(Method method){
        System.out.println("[Info] " + method.getName() + "方法被调用");
    }
}
```

Client类：

```java
package jdkproxy;

public class DynamicClient {
    public static void main(String[] args) {
        //真实角色
        UserServiceImpl userServiceImpl = new UserServiceImpl();
        //代理角色，不存在
        DynamicProxy dynamicProxy = new DynamicProxy();
        //设置要代理的对象
        dynamicProxy.setUserService((UserService) userServiceImpl);

        //动态生成代理类
        UserService proxy = (UserService) dynamicProxy.getProxy();
        proxy.add();
        proxy.delete();
        proxy.update();
        proxy.query();
    }
}
```

# 动态代理在反序列化中的作用

反序列化漏洞是需要一个入口类的。

假设存在一个能够利用的类为B.f，比如Runtime.exec。

将入口类定义为A，理想的情况是A[O] -> O.f，那么我们将传进去的参数`O`替换为`B`即可。但是这样的情况是极少的。

回到实战情况，比如我们的入口类`A`存在`O.abc`这个方法，也就是 A[O] -> O.abc；而 O 呢，如果是一个动态代理类，`O`的`invoke`方法里存在`.f`的方法，便可以漏洞利用了，我们还是展示一下。

```
A[O] -> O.abc
O[O2] invoke -> O2.f // 此时将 B 去替换 O2（动态代理类不管外面调什么方法都会调用它的invoke
最后  ---->
O[B] invoke -> B.f // 达到漏洞利用效果
```

动态代理在反序列化当中的利用和`readObject`是异曲同工的。

`readObject`方法在反序列化当中会被自动执行。
而`invoke`方法在动态代理当中会自动执行。

具体应用在后续的CC第一条链上，等过两天空了再看。



























# 参考链接

1. https://www.freebuf.com/articles/web/335236.html
2. https://www.bilibili.com/video/BV16h411z7o9?p=3