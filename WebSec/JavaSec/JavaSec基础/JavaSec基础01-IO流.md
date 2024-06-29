# Java-I/O流

## 基础

在变量、数组和对象中存储的数据是暂时存在的，程序结束后它们就会被丢失。想要永久地存储程序创建的数据，就需要将其保存在磁盘文件中，而只有数据被存储起来，在其他程序中才可以使用它们。Java的I/O技术可以将数据保存到文本文件、二进制文件甚至是ZIP压缩文件中，以达到永久性保存数据的要求。

Java语言定义了许多类专门负责各种方式的输入／输出，这些类都被放在java.io包中。其中，所有输入流类都是抽象类InputStream（字节输入流）或抽象类Reader（字符输入流）的子类；而所有输出流都是抽象类OutputStream（字节输出流）或抽象类Writer（字符输出流）的子类。

### 输入流

InputStream类是字节输入流的抽象类，它是所有字节输入流的父类。InputStream类的具体层次结构如下图所示。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231024112617.png)

该类中所有方法遇到错误时都会引发IOException异常。下面是对该类中的一些方法的简要说明：

- read()方法：从输入流中读取数据的下一个字节。返回0～255的int字节值。如果因为已经到达流末尾而没有可用的字节，则返回值为−1。
- read(byte[] b)：从输入流中读入一定长度的字节，并以整数的形式返回字节数。
- mark(int readlimit)方法：在输入流的当前位置放置一个标记，readlimit参数告知此输入流在标记位置失效之前允许读取的字节数。
- reset()方法：将输入指针返回到当前所做的标记处。
- skip(long n)方法：跳过输入流上的n个字节并返回实际跳过的字节数。
- markSupported()方法：如果当前流支持mark()/reset()操作就返回true。
- close方法：关闭此输入流并释放与该流关联的所有系统资源。

Java中的字符是Unicode编码，是双字节的。InputStream类是用来处理字节的，并不适合处理字符文本。Java为字符文本的输入专门提供了一套单独的类，即Reader类，但Reader类并不是InputStream类的替换者，只是在处理字符串时简化了编程。Reader类是字符输入流的抽象类，所有字符输入流的实现都是它的子类。Reader类的具体层次结构如下图所示。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231024121824.png)

### 输出流

OutputStream类是字节输出流的抽象类，此抽象类是表示输出字节流的所有类的超类。OutputStream类的具体层次如下图所示。

![image-20231024123146463](C:\Users\86158\AppData\Roaming\Typora\typora-user-images\image-20231024123146463.png)

OutputStream类中的所有方法均返回void，在遇到错误时会引发IOException异常。下面是对OutputStream类中的一些方法的简要说明：

- write(int b)方法：将指定的字节写入此输出流。
- write(byte[] b)方法：将b个字节从指定的byte数组写入此输出流。
- write(byte[] b,int off,int len)方法：将指定byte数组中从偏移量off开始的len个字节写入此输出流。
- flush()方法：彻底完成输出并清空缓存区。
- close()方法：关闭输出流。

Writer类是字符输出流的抽象类，所有字符输出类的实现都是它的子类。Writer类的层次结构如下图所示。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231024123535.png)



### File类

File类是java.io包中唯一代表磁盘文件本身的类。File类定义了一些与平台无关的方法来操作文件，可以通过调用File类中的方法，实现创建、删除、重命名文件等操作。File类的对象主要用来获取文件本身的一些信息，如文件所在的目录、文件的长度、文件读写权限等。数据流可以将数据写入文件中，文件也是数据流最常用的数据媒体。

```java
import java.io.File;

public class FileTest {
    public static void main(String[] args) {
        File file = new File("E:\\研究生\\Java\\File\\1.txt");
        if(file.exists()){
            file.delete();
            System.out.println("文件已删除！");
        }else{
            try{
                file.createNewFile();
                System.out.println("文件已创建！");
            }catch(Exception e){
                e.printStackTrace();
            }
        }
    }
}

```



### 获取文件信息

File类提供了很多方法以获取文件本身的信息，其中常用的方法如下表所示。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231024132535.png)

### 文件输入/输出流

#### FileInputStream与FileOutputStream类

FileInputStream类与FileOutputStream类都用来操作磁盘文件。如果用户的文件读取需求比较简单，则可以使用FileInputStream类，该类继承自InputStream类。FileOutputStream类与FileInputStream类对应，提供了基本的文件写入能力。FileOutputStream类是OutputStream类的子类。FileInputStream类常用的构造方法如下：

FileInputStream(String name)。

FileInputStream(File file)。

第一个构造方法使用给定的文件名name创建一个FileInputStream对象，第二个构造方法使用File对象创建FileInputStream对象。第一个构造方法比较简单，但第二个构造方法允许在把文件连接输入流之前对文件做进一步分析。

使用FileOutputStream类和FileInputStream类，向File目录的1.txt文件中写入一句话，然后再读取出来输出在控制台上：

```java
import java.io.File;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;

public class FileInputStreamTest {
    public static void main(String[] args) {
        File file = new File("E:\\研究生\\Java\\File\\1.txt");
        try{
            FileOutputStream out = new FileOutputStream(file);
            byte buy[] = "原神，启动！".getBytes();
            out.write(buy);
            out.close();
        }catch (IOException e){
            e.printStackTrace();
        }
        try{
            java.io.FileInputStream in = new java.io.FileInputStream(file);
            byte byt[] = new byte[1024];
            int len = in.read(byt);
            System.out.println("文件中的信息是：" + new String(byt, 0, len));
            in.close();
        }catch (IOException e){
            e.printStackTrace();
        }
    }
}
```

使用FileOutputStream类向文件中写入数据与使用FileInputStream类从文件中将内容读出来，都存在一点不足，即这两个类都只提供了对字节或字节数组的读取方法。由于汉字在文件中占用两个字节，如果使用字节流，读取不好可能会出现乱码现象，此时采用字符流FileReader类或FileWriter类即可避免这种现象。FileReader类和FileWriter类对应了FileInputStream类和FileOutputStream类。FileReader类顺序地读取文件，只要不关闭流，每次调用read()方法就顺序地读取源中其余的内容，直到源的末尾或流被关闭。

```java
import java.io.*;

public class FileWriterTest {
    public static void main(String[] args) {
        File file = new File("E:\\研究生\\Java\\File\\1.txt");
        try{
            FileWriter fw = new FileWriter(file);
            String word = "Apex Lengends,Start!";
            fw.write(word);
            fw.close();
        }catch (IOException e){
            e.printStackTrace();
        }
        try{
            FileReader fr = new FileReader(file);
            char ch[] = new char[1024];
            int len = fr.read(ch);
            System.out.println("文件中的信息是：" + new String(ch, 0, len));
            fr.close();
        }catch (IOException e){
            e.printStackTrace();
        }
    }
}
```

### 带缓存的输入/输出流

#### BufferedInputStream与BufferedOutputStream类

使用BufferedOutputStream类输出信息和仅用OutputStream类输出信息完全一样，只不过BufferedOutputStream有一个flush()方法用来将缓存区的数据强制输出完。

flush()方法就是用于即使在缓存区没有满的情况下，也将缓存区的内容强制写入外设，习惯上称这个过程为刷新。flush()方法只对使用缓存区的OutputStream类的子类有效。当调用close()方法时，系统在关闭流之前，也会将缓存区中的信息刷新到磁盘文件中。

#### BufferedReader与BufferedWriter类

BufferedReader类与BufferedWriter类分别继承Reader类与Writer类。这两个类同样具有内部缓存机制，并能够以行为单位进行输入／输出。根据BufferedReader类的特点，可以总结出如下图所示的带缓存的字符数据读取文件的过程。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231025131240.png)

BufferedReader类常用的方法如下：

- read()方法：读取单个字符。
- readLine()方法：读取一个文本行，并将其返回为字符串。若无数据可读，则返回null。BufferedWriter类中的方法都返回void。常用的方法如下：
- write(String s, int off, int len)方法：写入字符串的某一部分
- flush()方法：刷新该流的缓存。
- newLine()方法：写入一个行分隔符。

在使用BufferedWriter类的Write()方法时，数据并没有立刻被写入输出流，而是首先进入缓存区中。如果想立刻将缓存区中的数据写入输出流，一定要调用flush()方法。

使用BufferedReader类和BufferedWriter类，向File目录的1.txt文件中写入多行内容，然后再读取出来输出在控制台上。

```java
import java.io.*;

public class BufferedReaderTest {
    public static void main(String[] args) {
        String content[] = {"原神","Apex Lengends","OverWatch"};
        File file = new File("E:\\研究生\\Java\\File\\1.txt");
        try{
            FileWriter fw = new FileWriter(file);
            BufferedWriter bw = new BufferedWriter(fw);
            for(int k =0;k< content.length;k++){
                bw.write(content[k]);
                bw.newLine();
            }
            bw.close();
            fw.close();
        }catch(IOException e){
            e.printStackTrace();
        }
        try{
            FileReader fr =new FileReader(file);
            BufferedReader br = new BufferedReader(fr);
            String tmp =null;
            int i = 1;
            while((tmp = br.readLine())!=null){
                System.out.println("第" + i + "行是：" + tmp);
                i++;
            }
            br.close();
            fr.close();
        }catch(IOException e){
            e.printStackTrace();
        }
    }
}
```

### 数据输入/输出流

数据输入／输出流（DataInputStream类与DataOutputStream类）允许应用程序以与机器无关的方式从底层输入流中读取基本Java数据类型。也就是说，当读取一个数据时，不必再关心这个数值应当是哪种字节。

DataOutputStream类提供了将字符串、double数据、int数据、boolean数据写入文件的方法。其中，将字符串写入文件的方法有3种，分别是writeBytes(String s)、writeChars(String s)、writeUTF(String s)。由于Java中的字符是Unicode编码，是双字节的，writeBytes()方法只是将字符串中的每一个字符的低字节内容写入目标设备中；而writeChars()方法将字符串中的每一个字符的两个字节的内容都写到目标设备中；writeUTF()方法将字符串按照UTF编码后的字节长度写入目标设备，然后才是每一个字节的UTF编码。DataInputStream类只提供了一个readUTF()方法返回字符串。这是因为要在一个连续的字节流读取一个字符串，如果没有特殊的标记作为一个字符串的结尾，并且不知道这个字符串的长度，就无法知道读取到什么位置才是这个字符串的结束。DataOutputStream类中只有writeUTF()方法向目标设备中写入字符串的长度，所以也能准确地读回写入字符串。

### 对象序列化输入/输出流

ObjectOutputStream类的对象用于序列化一个对象。ObjectInputStream类的对象用于反序列化一个对象。

一个类只有实现Serializable或Externalizable接口，才能被序列化或反序列化。例如，序列化一个Book类对象的代码如下；

```java
import java.io.Serializable
public class Book implements Serializable{
    
}
```

实现Externalizable接口能够更好地控制从输入流中读取对象和向输入流中写入对象，因为Externalizable接口继承了Serializable接口。当从输入流读取一个对象时，需要调用readExternal()方法；当向输出流中写入一个对象时，需要调用writeExternal()方法。

Book类实现Externalizable接口的代码如下：

```java
import java.io.Externalizable
import java.io.IOException
import java.io.ObjectInput
import java.io.ObjectOutput
    
public class Book implements Externalizable{
    public void readExternal(ObjectInput in) throws IOException,ClassNotFoundException{
        
    }
    public void writeExternal(ObjectInput out) throws IOException{
        
    }
}
```

#### 序列化对象

序列化一个Book类对象需要执行如下步骤：

- 创建ObjectOutputStream类的对象，并将对象保存到book.ser文件中。

- 将对象保存到ByteArrayOutputStream类中，并构造一个对象输出流。

- 使用ObjectOutputStream类的writeObject()方法序列化对象。

- 关闭对象输出流

  

编写Book类：

```java
package ObjectOutputStreamTest;

import java.io.Serializable;

public class Book implements Serializable {
    private String name;
    private double price;
    public Book(String name,double price){
        this.name = name;
        this.price = price;
    }

    @Override
    public String toString(){
        return "Name: " + this.name + ",price: " + this.price;
    }
}

```

序列化Book类：

```java
package ObjectOutputStreamTest;

import java.io.*;
//序列化
public class BookOutTest {
    public static void main(String[] args) {
        Book book1 = new Book("原神",30);
        Book book2 = new Book("守望先锋",0.01);
        File fileObject = new File("book.ser");
        try(ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(fileObject))){
            oos.writeObject(book1);
            oos.writeObject(book2);
            System.out.println(book1);
            System.out.println(book2);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}


```

反序列化Book类：

```java
package ObjectOutputStreamTest;

import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;
import java.io.ObjectInputStream;

//反序列化
public class BookInTest {
    public static void main(String[] args) {
        File fileObject = new File("book.ser");
        try(ObjectInputStream ois = new ObjectInputStream(new FileInputStream(fileObject))){
            Book book1 = (Book) ois.readObject();
            Book book2 = (Book) ois.readObject();
            System.out.println(book1);
            System.out.println(book2);
        }catch(Exception e){
            e.printStackTrace();
        }
    }
}

```

