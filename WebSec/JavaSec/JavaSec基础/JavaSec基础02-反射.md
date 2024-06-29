参考链接：https://wx.zsxq.com/dweb2/index/group/2212251881

Java安全可以从反序列化漏洞开始说起，反序列化漏洞⼜可以从反射开始说起。

# 基础

这个师傅写的不错：https://drun1baby.top/2022/05/20/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%9F%BA%E7%A1%80%E7%AF%87-02-Java%E5%8F%8D%E5%B0%84%E4%B8%8EURLDNS%E9%93%BE%E5%88%86%E6%9E%90/#0x01-%E5%89%8D%E8%A8%80

通过Java反射机制，可以在程序中访问已经装载到JVM中的Java对象的描述，实现访问、检测和修改描述Java对象本身信息的功能。Java反射机制的功能十分强大，在java.lang.reflect包中提供了对该功能的支持。

众所周知，所有Java类均继承了Object类，在Object类中定义了一个getClass()方法，该方法返回一个类型为Class的对象。例如下面的代码：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231025164759.png)

利用Class类的对象textFieldC，可以访问用来返回该对象的textField对象的描述信息。可以访问的主要描述信息如下表所示。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231025165439.png)

注：在通过getFields()方法和getMethods()方法依次获得权限为public的成员变量和方法时，将包含从超类中继承到的成员变量和方法；而通过getDeclaredFields()方法和getDeclaredMethods()方法只是获得在本类中定义的所有成员变量和方法。

 ## 访问构造方法

在通过下列一组方法访问构造方法时，将返回Constructor类型的对象或数组。每个Constructor对象代表一个构造方法，利用Constructor对象可以操纵相应的构造方法：

- getConstructors()
- getConstructor(Class<?>...parameterTypes)
- getDeclaredConstructors()
- getDeclaredConstructor(Class<?>...parameterTypes)。

如果是访问指定的构造方法，需要根据该构造方法的入口参数的类型来访问。例如，访问一个入口参数类型依次为String型和int型的构造方法，通过下面两种方式均可实现：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231025171712.png)

Constructor类中提供的常用方法如下表所示。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231025171829.png)

通过java.lang.reflect.Modifier类可以解析出getModifiers()方法的返回值所表示的修饰符信息，在该类中提供了一系列用来解析的静态方法，既能查看是否被指定的修饰符修饰，又能以字符串的形式获得所有修饰符。Modifier类常用静态方法如下表所示。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231025172043.png)

例如，判断对象constructor所代表的构造方法是否被private修饰，以及以字符串形式获得该构造方法的所有修饰符的典型代码如下：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231025172634.png)

例1：反射一个类的所有构造方法。

创建一个Demo1类，在该类中声明一个String型成员变量和3个int型成员变量，并提供3个构造方法。具体代码如下：

```java
package fanshe;

public class Demo1 {
    String s;
    int i1,i2,i3;
    private Demo1(){

    }
    protected Demo1(String s,int i1){
        this.s = s;
        this.i1 = i1;
    }
    public Demo1(String... strings) throws NumberFormatException{
        if(0 < strings.length){
            i1 = Integer.valueOf(strings[0]);
        }
        if(1 < strings.length){
            i2 = Integer.valueOf(strings[1]);
        }
        if(2 < strings.length){
            i3 = Integer.valueOf(strings[2]);
        }
    }
    public void print(){
        System.out.println("s=" + s);
        System.out.println("i1=" + i1);
        System.out.println("i2=" + i2);
        System.out.println("i3=" + i3);
    }
}
```

然后编写测试类ConstructorDemo1，在该类中通过反射访问com.mr.Demo1类中的所有构造方法，并将该构造方法是否允许带有可变数量的参数、入口参数类型和可能抛出的异常类型信息输出到控制台。具体代码如下：

```java
package fanshe;

import java.lang.reflect.Constructor;

public class ConstructorDemo1 {
    public static void main(String[] args) {
        Demo1 d1 = new Demo1("10","20","30");
        //<? extends Number>表示上边界限定，泛型参数只能是Number及其子类，一旦实例化其他参数类型则会报错
        //<? super Number>表示下边界限定，泛型参数只能是Number和它的父类。
        Class<? extends Demo1> Demo1Class = d1.getClass();
        //获得所有构造方法
        Constructor[] declaredConstructers = Demo1Class.getDeclaredConstructors();
        for (int i = 0;i< declaredConstructers.length;i++){
            Constructor<?> constructor = declaredConstructers[i];
            System.out.println("查看是否允许带有可变数量的参数：" + constructor.isVarArgs());
            System.out.println("该构造方法的入口参数类型依次为：");
            Class[] parameterTypes = constructor.getParameterTypes();
            for (int j = 0;j < parameterTypes.length;j++){
                System.out.println(" " + parameterTypes[j]);
            }
            System.out.println("该构造方法可能抛出的异常类型为：");
            Class[] exceptionTypes = constructor.getExceptionTypes();
            for (int j =0;j<exceptionTypes.length;j++){
                System.out.println(" " + exceptionTypes[j]);
            }
            Demo1 d2 = null;
            while(d2 == null){
                try{//如果构造方法的访问权限为private或protected，则抛出异常，不允许访问
                    if(i == 2)//通过执行默认没有参数的构造方法创建对象
                        d2 = (Demo1) constructor.newInstance();
                    else if(i == 1)
                        //通过执行具有两个参数的构造方法创建对象
                        d2 = (Demo1) constructor.newInstance("7",5);
                    else {//通过执行具有可变数量参数的构造方法创建对象
                        Object[] parameters = new Object[]{new String[]{"100", "200", "300"}};
                        d2 = (Demo1) constructor.newInstance(parameters);
                    }
                }catch (Exception e){
                    System.out.println("在创建对象时抛出异常，下面执行setAccessible()方法");
                    constructor.setAccessible(true);
                }
            }
            if(d2!=null){
                d2.print();
                System.out.println();
            }
        }
    }
}
```

运行结果如下:

```
查看是否允许带有可变数量的参数：true
该构造方法的入口参数类型依次为：
 class [Ljava.lang.String;
该构造方法可能抛出的异常类型为：
 class java.lang.NumberFormatException
s=null
i1=100
i2=200
i3=300

查看是否允许带有可变数量的参数：false
该构造方法的入口参数类型依次为：
 class java.lang.String
 int
该构造方法可能抛出的异常类型为：
s=7
i1=5
i2=0
i3=0

查看是否允许带有可变数量的参数：false
该构造方法的入口参数类型依次为：
该构造方法可能抛出的异常类型为：
在创建对象时抛出异常，下面执行setAccessible()方法
s=null
i1=0
i2=0
i3=0
```



## 访问成员变量

在通过下列一组方法访问成员变量时，将返回Field类型的对象或数组。每个Field对象代表一个成员变量，利用Field对象可以操纵相应的成员变量：

- getFields()。
- getField(String name)。
- getDeclaredFields()。
- getDeclaredField(String name)。

如果是访问指定的成员变量，可以通过该成员变量的名称来访问。例如，访问一个名称为birthday的成员变量，访问方法如下：

```java
object. getDeclaredField("birthday");
```

Field类中提供的常用方法如下表所示。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231026133734.png)

例2：反射一个类的所有成员变量

创建一个Demo2类，在该类中依次声明一个int、float、boolean和String型的成员变量，并将它们设置为不同的访问权限。具体代码如下：

```java
package fanshe;

public class Demo2 {
    int i;
    public float f;
    protected boolean b;
    private String s;
}
```

然后通过反射访问com.mr.Demo2类中的所有成员变量，将成员变量的名称和类型信息输出到控制台，并分别将各个成员变量在修改前后的值输出到控制台。关键代码如下：

```java
package fanshe;

import java.lang.reflect.Field;

public class FieldDemo2 {
    public static void main(String[] args) {
        Demo2 example = new Demo2();
        Class exampleC = example.getClass();
        Field[] declaredFields = exampleC.getDeclaredFields(); //获得所有成员变量
        for (int i = 0;i<declaredFields.length;i++){           //遍历成员变量
            Field field = declaredFields[i];
            System.out.println("名称为：" + field.getName());   //获得成员变量名称并输出
            Class fieldType = field.getType();                //获得成员变量类型
            System.out.println("类型为：" + field.getType());
            boolean isTurn = true;
            while(isTurn){
                //如果该成员变量的访问权限为private或protected，则抛出异常，即不允许访问
                try{
                    isTurn = false;
                    //获得成员变量值
                    System.out.println("修改前的值为：" + field.get(example));
                    if (fieldType.equals(int.class)){
                        System.out.println("利用方法setInt()修改成员变量的值");
                        field.setInt(example,168);
                    }else if(fieldType.equals(float.class)){
                        System.out.println("利用方法setFloat()修改成员变量的值");
                        field.setFloat(example,99.9F);
                    }else if(fieldType.equals(boolean.class)) {
                        System.out.println("利用方法setFloat()修改成员变量的值");
                        field.setBoolean(example, true);
                    }
                    else{
                        System.out.println("利用方法set()修改成员变量的值");
                        field.set(example,"MWQ");
                    }
                    System.out.println("修改后的值为" + field.get(example));
                }catch (Exception e){
                    System.out.println("在设置变量成员时抛出异常，" + "下面执行setAccessible()方法");
                    field.setAccessible(true);
                    isTurn = true;
                }
            }
            System.out.println();
        }
    }
}
```

执行结果如下：

```
名称为：i
类型为：int
修改前的值为：0
利用方法setInt()修改成员变量的值
修改后的值为168

名称为：f
类型为：float
修改前的值为：0.0
利用方法setFloat()修改成员变量的值
修改后的值为99.9

名称为：b
类型为：boolean
修改前的值为：false
利用方法setFloat()修改成员变量的值
修改后的值为true

名称为：s
类型为：class java.lang.String
在设置变量成员时抛出异常，下面执行setAccessible()方法
修改前的值为：null
利用方法set()修改成员变量的值
修改后的值为MWQ
```

## 访问成员方法

在通过下列一组方法访问成员方法时，将返回Method类型的对象或数组。每个Method对象代表一个方法，利用Method对象可以操纵相应的方法：

- getMethods()。
- getMethod(String name, Class<?>...parameterTypes)。
- getDeclaredMethods()。
- getDeclaredMethod(String name, Class<?>...parameterTypes)。

如果是访问指定的方法，需要根据该方法的名称和入口参数的类型来访问。例如，访问一个名称为print、入口参数类型依次为String型和int型的方法，通过下面两种方式均可实现：

```
objectClass.getDeclaredMethod("print", String.class, int.class);
objectClass.getDeclaredMethod("print", new Class[] {String.class, int.class });
```

Method类中提供的常用方法如下表所示。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231026143400.png)

例3：反射一个类的所有成员方法。

创建一个Demo3类，并编写4个成员方法。具体代码如下：

```java
package fanshe;

public class Demo3 {
    static void staticMethod() {
        System.out.println("执行staticMethod()方法");
    }

    public int publicMethod(int i) {
        System.out.println("执行publicMethod()方法");
        return i * 100;
    }

    protected int protectedMethod(String s, int i) throws NumberFormatException {
        System.out.println("执行protectedMethod()方法");
        return Integer.valueOf(s) + i;
    }

    private String privateMethod(String... strings) {
        System.out.println("执行privateMethod()方法");
        StringBuffer stringBuffer = new StringBuffer();
        for (int i = 0; i < strings.length; i++) {
            stringBuffer.append(strings[1]);
        }
        return stringBuffer.toString();
    }
}
```

然后通过反射访问Demo3类中的所有方法，将各个方法的名称、入口参数类型、返回值类型等信息输出到控制台，并执行部分方法。关键代码如下：

```java
package fanshe;

import java.lang.reflect.*;

public class MethodDemo3 {
    public static void main(String[] args) {
        Demo3 demo = new Demo3();
        Class demoClass = demo.getClass();
        Method[] declaredMethods = demoClass.getDeclaredMethods();//获得所有方法
        for (int j = 0;j< declaredMethods.length;j++){
            Method method = declaredMethods[j];
            System.out.println("名称是：" + method.getName());
            System.out.println("是否允许带有可变数量的参数：" + method.isVarArgs());
            System.out.println("入口参数类型依次为");
            Class[] parameterTypes = method.getParameterTypes();
            for(int i = 0;i< parameterTypes.length;i++){
                System.out.println(" " + parameterTypes[i]);
            }
            System.out.println("返回值类型为：" + method.getReturnType());
            System.out.println("可能抛出的异常类型有：");
            Class[] exceptionTypes = method.getExceptionTypes();
            for (int i = 0;i< exceptionTypes.length;i++){
                System.out.println(" " + exceptionTypes[i]);
            }
            boolean isTurn = true;
            while(isTurn){
                //如果该成员变量的访问权限为private或protected，则抛出异常，即不允许访问
                try{
                    isTurn = false;
                    if ("staticMethod".equals(method.getName())){
                        method.invoke(demo);
                    }else if("publicMethod".equals(method.getName())){
                        System.out.println("返回值为：" + method.invoke(demo,168));
                    }else if("protectedMethod".equals(method.getName())) {
                        System.out.println("返回值为：" + method.invoke(demo,"7",5));
                    }else if("privateMethod".equals(method.getName())) {
                        Object[] parameters = new Object[]{new String[]{"M","W","Q"}};
                        System.out.println("返回值为：" + method.invoke(demo,parameters));
                    }
                }catch (Exception e){
                    System.out.println("在执行方法时抛出异常，" + "下面执行setAccessible()方法");
                    method.setAccessible(true);
                    isTurn = true;
                }
            }
            System.out.println();
        }
    }
}
```

运行结果如下：

```
名称是：staticMethod
是否允许带有可变数量的参数：false
入口参数类型依次为
返回值类型为：void
可能抛出的异常类型有：
执行staticMethod()方法

名称是：privateMethod
是否允许带有可变数量的参数：true
入口参数类型依次为
 class [Ljava.lang.String;
返回值类型为：class java.lang.String
可能抛出的异常类型有：
在执行方法时抛出异常，下面执行setAccessible()方法
执行privateMethod()方法
返回值为：MWQ

名称是：protectedMethod
是否允许带有可变数量的参数：false
入口参数类型依次为
 class java.lang.String
 int
返回值类型为：int
可能抛出的异常类型有：
 class java.lang.NumberFormatException
执行protectedMethod()方法
返回值为：12

名称是：publicMethod
是否允许带有可变数量的参数：false
入口参数类型依次为
 int
返回值类型为：int
可能抛出的异常类型有：
执行publicMethod()方法
返回值为：16800
```

# 正文

注：总结自p神的Java安全漫谈。

反射是⼤多数语⾔⾥都必不可少的组成部分，对象可以通过反射获取他的类，类可以通过反射拿到所有
⽅法（包括私有），拿到的⽅法可以调⽤，总之通过“反射”，我们可以将Java这种静态语⾔附加上动态
特性。  

Java虽不像PHP那么灵活，但其提供的“反射”功能，也是可以提供⼀些动态特性。⽐如，这样⼀段代码，在你不知道传⼊的参数值的时候，你是不知道他的作⽤是什么的：  

```java
public void execute(String className, String methodName) throws Exception {
	Class clazz = Class.forName(className);
	clazz.getMethod(methodName).invoke(clazz.newInstance());
}
```

上⾯的例⼦中，包含了几个在反射里极为重要的方法：

- 获取类的⽅法： forName
- 实例化类对象的⽅法： newInstance
- 获取函数的⽅法： getMethod
- 执⾏函数的⽅法： invoke

基本上，这几个方法包揽了Java安全里各种和反射有关的Payload 。

forName不是获取类的唯一途径，根据基础部分所学内容，通常来说一共有三种方式获取一个“类”，也就是java.lang.Class对象：

- obj.getClass() 如果上下文中存在某个类的实例 obj ，那么我们可以直接通过obj.getClass() 来获取它的类
- Test.class 如果你已经加载了某个类，只是想获取到它的 java.lang.Class 对象，那么就直接拿它的 class 属性即可。这个方法其实不属于反射。
- Class.forName 如果你知道某个类的名字，想获取到这个类，就可以使用 forName 来获取  

## forName

在安全研究中，我们使⽤反射的一大目的，就是绕过某些沙盒。比如，上下文中如果只有Integer类型的
数字，我们如何获取到可以执⾏命令的Runtime类呢？也许可以这样（伪代码）： 1.getClass().forName("java.lang.Runtime")  

forName有两个函数重载：

- Class<?> forName(String name)
- Class<?> forName(String name, **boolean** initialize, ClassLoader loader)  

第⼀个就是我们最常见的获取class的⽅式，其实可以理解为第⼆种方式的⼀个封装：

```java
Class.forName(className)
// 等于
Class.forName(className, true, currentLoader)
```

默认情况下， forName 的第⼀个参数是类名；第⼆个参数表示是否初始化；第三个参数就
是 ClassLoader 。  

ClassLoader 是什么呢？它就是⼀个“加载器”，告诉Java虚拟机如何加载这个类。关于这个点，后⾯还
有很多有趣的漏洞利⽤⽅法，这⾥先不展开说了。 Java默认的 ClassLoader 就是根据类名来加载类，
这个类名是类完整路径，如 java.lang.Runtime 

第二个参数initialize常常被人误解。其实在 forName 的时候，构造函数并不会执行，即使我们设
置 initialize=true。那么这个初始化究竟指什么呢？

可以将这个“初始化”理解为类的初始化。我们先来看看如下这个类：

```java
public class TrainPrint {
	{
		System.out.printf("Empty block initial %s\n", this.getClass());
	}
    
	static {
		System.out.printf("Static initial %s\n", TrainPrint.class);
	}
    
	public TrainPrint() {
		System.out.printf("Initial %s\n", this.getClass());
	}
}
```

上述三个初始化方法调用的顺序是static{}，{}，最后是构造函数。

其中， static {} 就是在“类初始化”的时候调用的，而{} 中的代码会放在构造函数的 super() 后⾯，但在当前构造函数内容的前⾯。所以说， forName 中的 initialize=true 其实就是告诉Java虚拟机是否执行”类初始化“。

那么，假设我们有如下函数，其中函数的参数name可控：  

```java
public void ref(String name) throws Exception {
	Class.forName(name);
}
```

我们就可以编写⼀个恶意类，将恶意代码放置在 static {} 中，从而执行：

```java
import java.lang.Runtime;
import java.lang.Process;

public class TouchFile {
	static {
		try {
			Runtime rt = Runtime.getRuntime();
			String[] commands = {"touch", "/tmp/success"};
			Process pc = rt.exec(commands);
			pc.waitFor();//等待该进程结束
		} catch (Exception e) {
			// do nothing
		}
	}
}
```

住执行命令的方法：

```
Runtime runtime = Runtime.getRuntime();
Process p = runtime.exec(cmd);
```

## 静态方法

在正常情况下，除了系统类，如果我们想拿到一个类，需要先 import 才能使用。而使用forName就不
需要，这样对于我们攻击者来说就十分有利，我们可以加载任意类。  

在Java中，$符号的作用是查找内部类。Java的普通类 C1 中支持编写内部类 C2 ，而在编译的时候，会生成两个文件： C1.class 和C1$C2.class ，我们可以把他们看作两个无关的类，通过 Class.forName("C1$C2") 即可加载这个内部类。

获得类以后，我们可以继续使用反射来获取这个类中的属性、方法，也可以实例化这个类，并调用方

法。

class.newInstance() 的作用就是调用这个类的无参构造函数，这个比较好理解。不过，我们有时候

在写漏洞利用方法的时候，会发现使用 newInstance 总是不成功，这时候原因可能是：

1. 你使用的类没有无参构造函数
2. 你使用的类构造函数是私有的  

最常见的情况就是 java.lang.Runtime，这个类在我们构造命令执行Payload的时候很常见，但
我们不能直接这样来执行命令：  

```java
Class clazz = Class.forName("java.lang.Runtime");
clazz.getMethod("exec", String.class).invoke(clazz.newInstance(), "id");
```

报错如下：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231030110709.png)

原因是Runtime类的构造方法是私有的。

为什么会有类的构造方法是私有的，难道他不想让用户使用这个类吗？这其实涉及到很常见的设计模式：“单例模式”。（有时候工厂模式也会写成类似）

比如，对于Web应用来说，数据库连接只需要建立一次，而不是每次用到数据库的时候再新建立一个连
接，此时作为开发者你就可以将数据库连接使用的类的构造函数设置为私有，然后编写一个静态方法来
获取：  

```Java
public class TrainDB {
	private static TrainDB instance = new TrainDB();
	
	public static TrainDB getInstance() {
		return instance;
	} 
	private TrainDB() {
		// 建立连接的代码...
	}
}
```

这样，只有类初始化的时候会执行一次构造函数，后面只能通过 getInstance 获取这个对象，避免建
立多个数据库连接。

回到正题，Runtime类就是单例模式，我们只能通过 Runtime.getRuntime() 来获取到 Runtime 对
象。我们将上述Payload进行修改即可正常执行命令了：  

```java
Class clazz = Class.forName("java.lang.Runtime");
        clazz.getMethod("exec", String.class).invoke(clazz.getMethod("getRuntime").invoke(clazz),"calc.exe");
```

这里用到了 getMethod 和 invoke 方法。  

getMethod 的作用是通过反射获取一个类的某个特定的公有方法。而学过Java的同学应该清楚，Java中
支持类的重载，我们不能仅通过函数名来确定一个函数。所以，在调用 getMethod 的时候，我们需要
传给他你需要获取的函数的参数类型列表。  

比如这里的 Runtime.exec 方法有6个重载：  

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231030120324.png)

我们使用最简单的，也就是第一个，它只有一个参数，类型是String，所以我们使用
getMethod("exec", String.class) 来获取 Runtime.exec 方法。  

invoke 的作用是执行方法，它的第一个参数是：

- 如果这个方法是一个普通方法，那么第一个参数是类对象
- 如果这个方法是一个静态方法，那么第一个参数是类  

这也比较好理解了，我们正常执行方法是 [1].method([2], [3], [4]...) ，其实在反射里就是
method.invoke([1], [2], [3], [4]...) 。  

所以我们将上述命令执行的Payload分解一下就是：  

```java
Class clazz = Class.forName("java.lang.Runtime");
Method execMethod = clazz.getMethod("exec",String.class);
Method getRuntimeMethod = clazz.getMethod("getRuntime");
Object runtime = getRuntimeMethod.invoke(clazz);
execMethod.invoke(runtime, "calc.exe");
```

## getConstructor

上一节讲了利用单例模式里的静态方法来实现反射，但遗留下来两个问题：

- 如果一个类没有无参构造方法，也没有类似单例模式里的静态方法，我们怎样通过反射实例化该类
  呢？
- 如果一个方法或构造方法是私有方法，我们是否能执行它呢？  

第一个问题，我们需要用到一个新的反射方法 getConstructor 。

和 getMethod 类似， getConstructor 接收的参数是构造函数列表类型，因为构造函数也支持重载，所以必须用参数列表类型才能唯一确定一个构造函数。

获取到构造函数后，我们使用 newInstance 来执行。

比如，我们常用的另一种执行命令的方式ProcessBuilder，我们使用反射来获取其构造函数，然后调用
start() 来执行命令：

```java
Class clazz = Class.forName("java.lang.ProcessBuilder");
((ProcessBuilder)clazz.getConstructor(List.class).newInstance(Arrays.asList("calc.exe"))).start();
```

ProcessBuilder有两个构造函数：

- public ProcessBuilder(List<String> command)
- public ProcessBuilder(String... command)

上面用到了第一个形式的构造函数，所以我在 getConstructor 的时候传入的是 List.class 。  

但是，我们看到，前面这个Payload用到了Java里的强制类型转换，有时候我们利用漏洞的时候（在表达式上下文中）是没有这种语法的。所以，我们仍需利用反射来完成这一步。  

其实用的就是前面讲过的知识：

```java
Class clazz = Class.forName("java.lang.ProcessBuilder")
clazz.getMethod("start").invoke(clazz.getConstructor(List.class).newInstance(Arrays.asList("calc.exe")));
```

通过 getMethod("start") 获取到start方法，然后 invoke 执行， invoke 的第一个参数就是
ProcessBuilder Object了。  

那么，如果我们要使用 public ProcessBuilder(String... command) 这个构造函数，需要怎样用反
射执行呢？  

这又涉及到Java里的可变长参数（varargs）了。正如其他语言一样，Java也支持可变长参数，就是当你
定义函数的时候不确定参数数量的时候，可以使用 ... 这样的语法来表示“这个函数的参数个数是可变
的”。
对于可变长参数，Java其实在编译的时候会编译成一个数组，也就是说，如下这两种写法在底层是等价
的（也就不能重载）：  

```java
public void hello(String[] names) {}
public void hello(String...names) {}
```

也由此，如果我们有一个数组，想传给hello函数，只需直接传即可：  

```java
String[] names = {"hello", "world"};
hello(names);
```

那么对于反射来说，如果要获取的目标函数里包含可变长参数，其实我们认为它是数组就行了。

所以，我们将字符串数组的类 String[].class 传给 getConstructor ，获取 ProcessBuilder 的第二
种构造函数：  

```java
Class clazz = Class.forName("java.lang.ProcessBuilder");
clazz.getConstructor(String[].class)
```

在调用 newInstance 的时候，因为这个函数本身接收的是一个可变长参数，我们传给ProcessBuilder 的也是一个可变长参数，二者叠加为一个二维数组，所以整个Payload如下：  

```java
Class clazz = Class.forName("java.lang.ProcessBuilder");
((ProcessBuilder)clazz.getConstructor(String[].class).newInstance(new String[][]{{"calc.exe"}})).start();
```

或

```java
Class clazz = Class.forName("java.lang.ProcessBuilder");
clazz.getMethod("start").invoke(clazz.getConstructor(String[].class).newInstance(new String[][]{{"calc.exe"}}));
```

## getDeclared

再说到第二个问题，如果一个方法或构造方法是私有方法，我们是否能执行它呢？  

这就涉及到 getDeclared 系列的反射了，与普通的 getMethod 、 getConstructor 区别是：

- getMethod 系列方法获取的是当前类中所有公共方法，包括从父类继承的方法
- getDeclaredMethod 系列方法获取的是当前类中“声明”的方法，是实在写在这个类里的，包括私
  有的方法，但从父类里继承来的就不包含了  

举个例子，前文我们说过Runtime这个类的构造函数是私有的，我们需要用 Runtime.getRuntime() 来
获取对象。其实现在我们也可以直接用 getDeclaredConstructor 来获取这个私有的构造方法来实例
化对象，进而执行命令：  

```java
Class clazz = Class.forName("java.lang.Runtime");
Constructor m = clazz.getDeclaredConstructor();
m.setAccessible(true);
clazz.getMethod("exec", String.class).invoke(m.newInstance(), "calc.exe");
```

这里使用了一个方法 setAccessible ，这个是必须的。我们在获取到一个私有方法后，必须用
setAccessible 修改它的作用域，否则仍然不能调用。  

# 其它

## 关于反射修改static、final字段的补充

https://drun1baby.top/2022/05/29/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%9F%BA%E7%A1%80%E7%AF%87-03-Java%E5%8F%8D%E5%B0%84%E8%BF%9B%E9%98%B6/#0x04-Java-%E5%8F%8D%E5%B0%84%E4%BF%AE%E6%94%B9-static-final-%E4%BF%AE%E9%A5%B0%E7%9A%84%E5%AD%97%E6%AE%B5

通过反射把final修饰符去掉：

```java
package com.yyds;

import java.lang.reflect.Field;
import java.lang.reflect.Modifier;

public class TestReflection {
    public static final String test = "abc";
    public static void main(String[] args) throws Exception{
        TestReflection testReflection = new TestReflection();
        Field test = testReflection.getClass().getDeclaredField("test");
        Field modifier = test.getClass().getDeclaredField("modifiers");
        modifier.setAccessible(true);
        modifier.setInt(test,test.getModifiers() & ~Modifier.FINAL);
        test.set(testReflection,"success");
        System.out.println(test.get(testReflection));
    }
}
```

