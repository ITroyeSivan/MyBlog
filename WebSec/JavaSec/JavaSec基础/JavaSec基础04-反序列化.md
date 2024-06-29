# 1. 概述

## 1.1 序列化与反序列化

**Java序列化**是指把Java对象转换为字节序列的过程；而**Java反序列化**是指把字节序列恢复为Java对象的过程。

序列化分为两大部分：序列化和反序列化。序列化是这个过程的第一部分，将数据分解成字节流，以便存储在文件中或在网络上传输。反序列化就是打开字节流并重构对象。对象序列化不仅要将基本数据类型转换成字节表示，有时还要恢复数据。恢复数据要求有恢复数据的对象实例。

## 1.2 为什么需要序列化与反序列化

用于Java进程间通信。一方面，发送方需要把这个Java对象转换为字节序列，然后在网络上传送；另一方面，接收方需要从字节序列中恢复出Java对象。

好处：其好处一是实现了数据的持久化，通过序列化可以把数据永久地保存到硬盘上（通常存放在文件里），二是，利用序列化实现远程通信，即在网络上传送对象的字节序列。

- 想把内存中的对象保存到一个文件中或者数据库中时候；
- 想用套接字在网络上传送对象的时候；
- 想通过RMI传输对象的时候

## 1.3 几种常见的序列化和反序列化协议

- XML&SOAP
  XML 是一种常用的序列化和反序列化协议，具有跨机器，跨语言等优点，SOAP（Simple Object Access protocol） 是一种被广泛应用的，基于 XML 为序列化和反序列化协议的结构化消息传递协议

- JSON（Javascript Object Notation）
- Protobuf

# 2. 序列化实现

只有实现了Serializable或者Externalizable接口的类的对象才能被序列化为字节序列。（不是则会抛出异常）

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231102144634.png)

Serializable 接口

是 Java 提供的序列化接口，它是一个空接口

```java
public interface Serializable {
}
```

Serializable 用来标识当前类可以被 ObjectOutputStream 序列化，以及被 ObjectInputStream 反序列化。

## 2.1 Serializable接口的基本使用

通过 ObjectOutputStream 将需要序列化数据写入到流中，因为 Java IO 是一种装饰者模式，因此可以通过 ObjectOutStream 包装 FileOutStream 将数据写入到文件中或者包装 ByteArrayOutStream 将数据写入到内存中。同理，可以通过 ObjectInputStream 将数据从磁盘 FileInputStream 或者内存 ByteArrayInputStream 读取出来然后转化为指定的对象即可。
![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231102150212.png)

Person类：

```java
package serialize;

import java.io.Serializable;

public class Person implements Serializable {
    private String name;
    private int age;

    public Person(){

    }

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}

```

序列化：

```java
package serialize;

import java.io.FileOutputStream;
import java.io.IOException;
import java.io.ObjectOutputStream;

public class SerializeTest {
    public static void serialize(Object obj) throws IOException {
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("ser.bin"));
        oos.writeObject(obj);
    }
    public static void main(String[] args) throws Exception{
        Person person = new Person("Tro3ye",22);
        serialize(person);
    }
}
```

反序列化：

```java
package serialize;

import java.io.*;

public class UnserializeTest {
    public static Object unserialize(String filename) throws IOException, ClassNotFoundException {
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream("ser.bin"));
        Object obj = ois.readObject();
        return obj;
    }
    public static void main(String[] args) throws Exception{
        Person person = (Person)unserialize("ser.bin");
        System.out.println(person);
    }
}
```

## 2.2 Serializable 接口的特点

1. 序列化类的属性没有实现 Serializable 那么在序列化就会报错

   ![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231102153430.png)

2. 在反序列化过程中，它的父类如果没有实现序列化接口，那么将需要提供无参构造函数来重新创建对象。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231103113359.png)

3. 一个实现 Serializable 接口的子类也是可以被序列化的。

4. 静态成员变量是不能被序列化

序列化是针对对象属性的，而静态成员变量是属于类的。

5. transient 标识的对象成员变量不参与序列化

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231103114239.png)

# 3. 安全问题

只要服务端反序列化数据，客户端传递类的readObject中代码会自动执行，给予攻击者在服务器上运行代码的能力。

可能的形式：

1. 入口类的readObject直接调用危险方法

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231103120855.png)

基本不可能出现的情况，因为没有人会在服务器上写危险代码。

2. 入口类参数包含可控类，该类有危险方法，readObject时调用。
3. 入口类参数中包含可控类，该类又调用其他有危险方法的类，readObject时调用。

4. 构造函数、静态代码块等类加载时隐式执行。

**产生漏洞的攻击路线**：

前提：继承Serializable

- 入口类 source （重写readObject 调用常见的函数 参数类型宽泛 最好是jdk自带）
- 调用链 gadget chain
- 执行类 sink RCE、SSRF





















# 参考链接

1.https://blog.csdn.net/mocas_wang/article/details/107621010

2.https://johnfrod.top/%e5%ae%89%e5%85%a8/java%e5%8f%8d%e5%ba%8f%e5%88%97%e5%8c%96%e6%bc%8f%e6%b4%9e-%e5%9f%ba%e7%a1%80%e7%af%87/

3.https://www.bilibili.com/video/BV16h411z7o9

4.https://drun1baby.top/2022/05/17/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%9F%BA%E7%A1%80%E7%AF%87-01-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%A6%82%E5%BF%B5%E4%B8%8E%E5%88%A9%E7%94%A8/

5.https://wx.zsxq.com/dweb2/index/group/2212251881