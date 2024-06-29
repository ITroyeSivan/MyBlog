# 什么是Class文件

Java字节码类文件（.class）是Java编译器编译Java源文件（.java）产生的“目标文件”。它是一种8位字节的二进制流文件， 各个数据项按顺序紧密的从前向后排列， 相邻的项之间没有间隙， 这样可以使得class文件非常紧凑， 体积轻巧， 可以被JVM快速的加载至内存， 并且占据较少的内存空间（方便于网络的传输）。

Java源文件在被Java编译器编译之后， 每个类（或者接口）都单独占据一个class文件， 并且类中的所有信息都会在class文件中有相应的描述， 由于class文件很灵活， 它甚至比Java源文件有着更强的描述能力。

class文件中的信息是一项一项排列的， 每项数据都有它的固定长度， 有的占一个字节， 有的占两个字节， 还有的占四个字节或8个字节， 数据项的不同长度分别用u1, u2, u4, u8表示， 分别表示一种数据项在class文件中占据一个字节， 两个字节， 4个字节和8个字节。 可以把u1, u2, u3, u4看做class文件数据项的“类型” 。

## Class文件的结构

一个典型的class文件分为：MagicNumber，Version，Constant_pool，Access_flag，This_class，Super_class，Interfaces，Fields，Methods 和Attributes这十个部分，用一个数据结构可以表示如下：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231130103339.png)

下面对class文件中的每一项进行详细的解释：

1、**magic**
在class文件开头的四个字节， 存放着class文件的魔数， 这个魔数是class文件的标志，他是一个固定的值： 0XCAFEBABE 。 也就是说他是判断一个文件是不是class格式的文件的标准， 如果开头四个字节不是0XCAFEBABE， 那么就说明它不是class文件， 不能被JVM识别。

2、**minor_version 和 major_version**
紧接着魔数的四个字节是class文件的此版本号和主版本号。
随着Java的发展， class文件的格式也会做相应的变动。 版本号标志着class文件在什么时候， 加入或改变了哪些特性。 举例来说， 不同版本的javac编译器编译的class文件， 版本号可能不同， 而不同版本的JVM能识别的class文件的版本号也可能不同， 一般情况下， 高版本的JVM能识别低版本的javac编译器编译的class文件， 而低版本的JVM不能识别高版本的javac编译器编译的class文件。 如果使用低版本的JVM执行高版本的class文件， JVM会抛出java.lang.UnsupportedClassVersionError 。具体的版本号变迁这里不再讨论， 需要的读者自行查阅资料。

3、**constant_pool**
在class文件中， 位于版本号后面的就是常量池相关的数据项。 常量池是class文件中的一项非常重要的数据。 常量池中存放了文字字符串， 常量值， 当前类的类名， 字段名， 方法名， 各个字段和方法的描述符， 对当前类的字段和方法的引用信息， 当前类中对其他类的引用信息等等。 常量池中几乎包含类中的所有信息的描述， class文件中的很多其他部分都是对常量池中的数据项的引用，比如后面要讲到的this_class, super_class, field_info, attribute_info等， 另外字节码指令中也存在对常量池的引用， 这个对常量池的引用当做字节码指令的一个操作数。此外，常量池中各个项也会相互引用。

常量池是一个类的结构索引，其它地方对“对象”的引用可以通过索引位置来代替，我们知道在程序中一个变量可以不断地被调用，要快速获取这个变量常用的方法就是通过索引变量。这种索引我们可以直观理解为“内存地址的虚拟”。我们把它叫静态池的意思就是说这里维护着经过编译“梳理”之后的相对固定的数据索引，它是站在整个JVM（进程）层面的共享池。

class文件中的项constant_pool_count的值为1, 说明每个类都只有一个常量池。 常量池中的数据也是一项一项的， 没有间隙的依次排放。常量池中各个数据项通过索引来访问， 有点类似与数组， 只不过常量池中的第一项的索引为1, 而不为0, 如果class文件中的其他地方引用了索引为0的常量池项， 就说明它不引用任何常量池项。class文件中的每一种数据项都有自己的类型， 相同的道理，常量池中的每一种数据项也有自己的类型。 常量池中的数据项的类型如下表：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231130104559.png)

每个数据项叫做一个XXX_info项，比如，一个常量池中一个CONSTANT_Utf8类型的项，就是一个CONSTANT_Utf8_info 。除此之外， 每个info项中都有一个标志值（tag），这个标志值表明了这个常量池中的info项的类型是什么， 从上面的表格中可以看出，一个CONSTANT_Utf8_info中的tag值为1，而一个CONSTANT_Fieldref_info中的tag值为9 。

Java程序是动态链接的， 在动态链接的实现中， 常量池扮演者举足轻重的角色。 除了存放一些字面量之外， 常量池中还存放着以下几种符号引用：

（1） 类和接口的全限定名

（2） 字段的名称和描述符

（3） 方法的名称和描述符

我们有必要先了解一下class文件中的特殊字符串， 因为在常量池中， 特殊字符串大量的出现，这些特殊字符串就是上面说的全限定名和描述符。对于常量池中的特殊字符串的了解，可以参考此文档：[Java class文件格式之特殊字符串_动力节点Java学院整理](http://www.jb51.net/article/116313.htm)

4、**access_flag** 保存了当前类的访问权限

5、**this_class** 保存了当前类的全局限定名在常量池里的索引

6、**super class** 保存了当前类的父类的全局限定名在常量池里的索引

7、**interfaces** 保存了当前类实现的接口列表，包含两部分内容：interfaces_count 和interfaces[interfaces_count]
interfaces_count 指的是当前类实现的接口数目
interfaces[] 是包含interfaces_count个接口的全局限定名的索引的数组

8、**fields** 保存了当前类的成员列表，包含两部分的内容：fields_count 和 fields[fields_count]
fields_count是类变量和实例变量的字段的数量总和。
fileds[]是包含字段详细信息的列表。

9、**methods** 保存了当前类的方法列表，包含两部分的内容：methods_count和methods[methods_count]
methods_count是该类或者接口显示定义的方法的数量。
method[]是包含方法信息的一个详细列表。

10、**attributes** 包含了当前类的attributes列表，包含两部分内容：attributes_count 和 attributes[attributes_count]
class文件的最后一部分是属性，它描述了该类或者接口所定义的一些属性信息。attributes_count指的是attributes列表中包含的attribute_info的数量。
属性可以出现在class文件的很多地方，而不只是出现在attributes列表里。如果是attributes表里的属性，那么它就是对整个class文件所对应的类或者接口的描述；如果出现在fileds的某一项里，那么它就是对该字段额外信息的描述；如果出现在methods的某一项里，那么它就是对该方法额外信息的描述。

## 示例

 新建一个java文件，Hello.java，具体内容如下：

```java
public class Hello{
  private int test;
  public int test(){
        return test;
    }
}
```

然后再通过javac命令将此java文件编译成class文件：

javac /d/class_file_test/Hello.java，编译之后的class文件十六进制结果如下所示，可以用UltraEdit等十六进制编辑器打开，得到：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231130125441.png)

接下来我们就按照class文件的格式来分析上面的一串数字，还是按照之前的顺序来：

1. **magic**:
   `CA FE BA BE` ，代表该文件是一个字节码文件，我们平时区分文件类型都是通过后缀名来区分的，不过后缀名是可以随便修改的，所以仅靠后缀名不能真正区分一个文件的类型。区分文件类型的另个办法就是magic数字，JVM 就是通过 CA FE BA BE 来判断该文件是不是class文件

2. **version字段**：
   `00 00 00 34`，前两个字节00是minor_version，后两个字节0034是major_version字段，对应的十进制值为52，也就是说当前class文件的主版本号为52，次版本号为0。下表是jdk 1.6 以后对应支持的 Class 文件版本号：

   ![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231130125627.png)

3. **常量池，constant_pool:**
   3.1. `constant_pool_count`
   紧接着version字段下来的两个字节是：`00 12`代表常量池里包含的常量数目，因为字节码的常量池是从1开始计数的，这个常量池包含17个（0x0012-1）常量。

   3.2.**constant_pool**
   接下来就是分析这17个常量:

   3.2.1. 第一个变量 `0a 00 04 00 0e`
   首先，紧接着constant_pool_count的第一个字节0a（tag=10）根据上面的表格（文中第二张图片）

   ![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231130133301.png)

   可知，这表示的是一个CONSTANT_Methodref。CONSTANT_Methodref的结构如下：

   ```
   CONSTANT_Methodref_info {
             u1 tag;    //u1表示占一个字节
             u2 class_index;    //u2表示占两个字节
             u2 name_and_type_index;    //u2表示占两个字节
   }
   ```

   其中class_index表示该方法所属的类在常量池里的索引，name_and_type_index表示该方法的名称和类型的索引。常量池里的变量的索引从1开始。

   那么这个methodref结构的数据如下：

   ```
   0a  //tag  10表示这是一个CONSTANT_Methodref_info结构
   00 04 //class_index 指向常量池中第4个常量所表示的类
   00 0e  //name_and_type_index 指向常量池中第14个常量所表示的方法
   ```

   3.2.2. 第二个变量`09 00 03 00 0F`
   接着是第二个常量，它的tag是09，根据上面的表格可知，这表示的是一个CONSTANT_Fieldref的结构，它的结构如下：

   ```
    CONSTANT_Fieldref_info {
         u1 tag;
         u2 class_index;
         u2 name_and_type_index;
   }
   ```

   和上面的变量基本一致。

   ```
   09 //tag
   00 03 //指向常量池中第3个常量所表示的类
   00 0f //指向常量池中第15个常量所表示的变量
   ```

   3.2.3. 第三个变量 `07 00 10`
   tag为07表示是一个CONSTANT_Class变量，这个变量的结构如下：

   ```
    CONSTANT_Class_info {
             u1 tag;
             u2 name_index;
   }
   ```

   除了tag字段以外，还有一个name_index的值为`00 10`，即是指向常量池中第16个常量所表示的Class名称。

   3.2.4. 第四个变量`07 00 11`
   同上，也是一个CONSTANT_Class变量，不过，指向的是第17个常量所表示的Class名称。

   3.2.5. 第五个变量 `01 00 04 74 65 73 74`
   tag为1，表示这是一个CONSTANT_Utf8结构，这种结构用UTF-8的一种变体来表示字符串，结构如下所示：

   ```
   CONSTANT_Utf8_info {
                  u1 tag;
                  u2 length;
                  u1 bytes[length];
   }
   ```

   其中length表示该字符串的字节数，bytes字段包含该字符串的二进制表示。

   ```
   01 //tag  1表示这是一个CONSTANT_Utf8结构
   00 04 //表示这个字符串的长度是4字节,也就是后面的四个字节74 65 73 74
   74 65 73 74 //通过ASCII码表转换后，表示的是字符串“test”
   ```

   接下来的8个变量都是字符串，这里就不具体分析了。

   3.2.6. 第十四个常量 `0c 00 07 00 08`
   tag为0c，表示这是一个CONSTANT_NameAndType结构，这个结构用来描述一个方法或者成员变量。具体结构如下：

   ```
   CONSTANT_NameAndType_info {
             u1 tag;
             u2 name_index;
             u2 descriptor_index;
   }
   ```

   name_index表示的是该变量或者方法的名称，这里的值是0007，表示指向第7个常量，即是`<init>`。

   descriptor_index指向该方法的描述符的引用，这里的值是0008，表示指向第8个常量，即是`()V`，由前面描述符的语法可知，这个方法是一个无参的，返回值为void的方法。

   综合两个字段，可以推出这个方法是`void <init>()`。也即是指向这个NameAndType结构的Methodref的方法名为`void <init>()`，也就是说第一个常量表示的是`void <init>()`方法，这个方法其实就是此类的默认构造方法。

   3.2.7. 第十五个常量也是一个CONSTANT_NameAndType，表示的方法名为“int test()”，第2个常量引用了这个NameAndType，所以第二个常量表示的是“int test()”方法。

   3.2.8. 第16和17个常量也是字符串，可以按照前面的方法分析。

   3.3. **完整的常量池**
   最后，通过以上分析，完整的常量池如下：

   ```
   00 12  常量池的数目 18-1=17
   0a 00 04 00 0e  方法：java.lang.Ojbect void <init>()
   09 00 03 00 0f   方法 ：Hello int test() 
   07 00 10  字符串：Hello
   07 00 11 字符串：java.lang.Ojbect
   01 00 04 74 65 73 74 字符串：test
   01 00 01 49  字符串：I
   01 00 06 3c 69 6e 69 74 3e 字符串：<init>
   01 00 03 28 29 56 字符串：()V
   01 00 04 43 6f 64 65 字符串：Code 
   01 00 0f 4c 69 6e 65 4e 75 6d 62 65 72 54 61 62 6c 65 字符串：LineNumberTable 
   01 00 03 28 29 49 字符串：()I
   01 00 0a 53 6f 75 72 63 65 46 69 6c 65 字符串：SourceFile
   01 00 0a 48 65 6c 6c 6f 2e 6a 61 76 61 字符串：Hello.java
   0c 00 07 00 08 NameAndType：<init> ()V
   0c 00 05 00 06 NameAndType：test I
   01 00 05 48 65 6c 6c 6f 字符串：Hello
   01 00 10 6a 61 76 61 2f 6c 61 6e 67 2f 4f 62 6a 65 63 74 字符串：java/lang/Object
   ```

   通过这样分析其实非常的累，我们只是为了了解class文件的原理才来一步一步分析每一个二进制字节码。JDK提供了现成的工具可以直接解析此二进制文件，即javap工具(在JDK的bin目录下)，我们通过javap命令来解析此class文件：

   ```
   javap -v -p -s -sysinfo -constants /d/class_file_test/Hello.class
   ```

   解析得到的结果为：

   ![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231130141205.png)

4. **access_flag(u2)**
   `00 21`这两个字节的数据表示这个变量的访问标志位，JVM对访问标示符的规范如下：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231130141748.png)

这个表里面无法直接查询到0021这个值，原因是0021=0020+0001，也就是表示当前class的access_flag是`ACC_PUBLIC|ACC_SUPER`。ACC_PUBLIC和代码里的public 关键字相对应。ACC_SUPER表示当用invokespecial指令来调用父类的方法时需要特殊处理。

5. **this_class(u2)** `00 03`
   this_class指向constant pool的索引值，该值必须是CONSTANT_Class_info类型，这里是3，即指向常量池中的第三项，即是“Hello”。

6. **super_class** `00 04`
   super_class存的是父类的名称在常量池里的索引，这里指向第四个常量，即是“java/lang/Object”。

7. **interfaces**

   interfaces包含interfaces_count和interfaces[]两个字段。因为这里没有实现接口，所以就不存在interfces选项，所以这里的interfaces_count为0（0000），所以后面的内容也对应为空

8. **fields**

```
00 01 fields count        //表示成员变量的个数，此处为1个
00 02 00 05 00 06 00 00   //成员变量的结构
```

每个成员变量对应一个field_info结构：

```
field_info {
          u2 access_flags; 0002
          u2 name_index; 0005
          u2 descriptor_index; 0006
          u2 attributes_count; 0000
          attribute_info attributes[attributes_count];
}
```

access_flags为0002，即是ACC_PRIVATE
name_index指向常量池的第五个常量，为“test”
descriptor_index指向常量池的第6个常量为“I”
三个字段结合起来，说明这个变量是”private int test”。
接下来的是attribute字段，用来描述该变量的属性，因为这个变量没有附加属性，所以attributes_count为0，attribute_info为空。

9. **methods**

​		`00 02 00 01 00 07 00 08 00 01 00 09 ...`
​		最前面的2个字节是method_count
​		method_count：`00 02`，为什么会有两个方法呢？我们明明只写了一个方法，这是因为JVM 会自		动生成一个`<init>`方法，这个是类的默认构造方法。

接下来的内容是两个`method_info`结构：

```
method_info {
     u2 access_flags;
     u2 name_index;
     u2 descriptor_index;
     u2 attributes_count;
     attribute_info attributes[attributes_count];
}
```

前三个字段和field_info一样，可以分析出第一个方法是“public void ()”

```
00 01 ACC_PUBLIC
00 07  <init>
00 08  V()
```

接下来是attribute字段，也即是这个方法的附加属性，这里的attributes_count =1，也即是有一个属性。
每个属性的都是一个attribute_info结构，如下所示：

```
attribute_info {
     u2 attribute_name_index;
     u4 attribute_length;
     u1 info[attribute_length];
}
```

JVM预定义了部分attribute，但是编译器自己也可以实现自己的attribute写入class文件里，供运行时使用。不同的attribute通过attribute_name_index来区分。JVM规范里对以下attribute进行了预定义：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231130142412.png)

10. **attributes**

    最后剩下的内容是attributes，这里的attributes表示整个class文件的附加属性，不过结构还是和前面的attribute保持一致。

```
00 01
```

​    	表示有一个attribute。

​		Attribute结构如下：	

```
SourceFile_attribute {
     u2 attribute_name_index;
     u4 attribute_length;
     u2 sourcefile_index;
}
```

​		attribute_name_index为000c，指向第12个常量，为SourceFile，说明这个属性是Source
​		attribute_length为00000002
​		sourcefile_index为000d，表示指向常量池里的第13个常量，为`Hello.java`。
​		这个属性表明当前的class文件是从Hello.java文件编译而来。

# 什么是Java的“字节码”

**Java 字节码**（英语：Java bytecode）是[Java虚拟机](https://zh.wikipedia.org/wiki/Java虚拟机)执行的一种[指令](https://zh.wikipedia.org/wiki/指令)格式。大多数[操作码](https://zh.wikipedia.org/w/index.php?title=操作码&action=edit&redlink=1)都是一个[字节](https://zh.wikipedia.org/wiki/字节)长，而有些操作需要参数，导致了有一些多字节的操作码。而且并不是所有可能的256个操作码都被使用；其中有51个操作码被保留做将来使用。除此之外，原始[Java平台](https://zh.wikipedia.org/wiki/Java平台)开发商，[太阳微系统](https://zh.wikipedia.org/wiki/昇陽微系統)，额外保留了3个代码永久不使用。

以下是ChatGPT4的介绍：

---

Java字节码是Java虚拟机（JVM）执行的中间代码。它是由Java编译器从Java源代码编译而来的一种平台无关的低级代码。字节码对于不同类型的JVM来说是通用的，这意味着它不依赖于任何特定的计算机体系结构。正因为如此，Java语言通常被描述为“一次编写，到处运行”（Write Once, Run Anywhere，WORA）。

以下是有关Java字节码的一些关键点：

1. **平台无关性**：Java字节码可以在任何安装了JVM的系统上运行，不论系统的体系结构是什么。
2. **编译过程**：当你编译一个Java程序时，`javac` 编译器将Java源代码转换成字节码（`.class` 文件）。这些字节码文件随后可以在任何JVM上执行。
3. **执行**：JVM负责解释或编译字节码为特定平台的机器码。这一步可以在运行时动态进行，也就是所谓的“即时编译”（Just-In-Time, JIT 编译）。
4. **安全性和效率**：字节码的设计考虑到了安全性和执行效率。由于它是由高级源代码编译而来的，因此包含了丰富的数据结构信息，这使得JVM能够执行各种优化。
5. **工具和库**：存在许多工具和库可以分析、修改、生成或操作Java字节码，使得Java平台更加灵活和强大。

总之，Java字节码是Java平台的核心组成部分，它使Java能够跨平台运行并保持较高的执行效率。

---

严格来说，Java字节码（ByteCode）其实仅仅指的是Java虚拟机执行使用的一类指令，通常被存储
在.class文件中。

众所周知，不同平台、不同CPU的计算机指令有差异，但因为Java是一门跨平台的编译型语言，所以这
些差异对于上层开发者来说是透明的，上层开发者只需要将自己的代码编译一次，即可运行在不同平台
的JVM虚拟机中。

甚至，开发者可以用类似Scala、Kotlin这样的语言编写代码，只要你的编译器能够将代码编译成.class文
件，都可以在JVM虚拟机中运行：  

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231128170230.png)

# 利用URLClassLoader加载远程class文件

Java的ClassLoader是用来加载字节码文件最基础的方法。它就是一个“加载器”，告诉Java虚拟机如何加载这个类。Java默认的ClassLoader就是根据类名来加载类，这个类名是类完整路径，如 java.lang.Runtime 。  

ClassLoader的概念不做深入分析，本文要说的是这个ClassLoader： URLClassLoader 。  

URLClassLoader 实际上是我们平时默认使用的 AppClassLoader 的父类，所以，我们解释
URLClassLoader 的工作过程实际上就是在解释默认的Java类加载器的工作流程。  

正常情况下，Java会根据配置项 sun.boot.class.path 和 java.class.path 中列举到的基础路径（这些路径是经过处理后的 java.net.URL 类）来寻找.class文件来加载，而这个基础路径有分为三种情况：  

- URL未以斜杠 / 结尾，则认为是一个JAR文件，使用 JarLoader 来寻找类，即为在Jar包中寻找.class文件
- URL以斜杠 / 结尾，且协议名是 file ，则使用 FileLoader 来寻找类，即为在本地文件系统中寻找.class文件
- URL以斜杠 / 结尾，且协议名不是 file ，则使用最基础的 Loader 来寻找类  

我们正常开发的时候通常遇到的是前两者，那什么时候才会出现使用 Loader 寻找类的情况呢？当然是
非 file 协议的情况下，最常见的就是 http 协议。  

我们可以使用HTTP协议来测试一下，看Java是否能从远程HTTP服务器上加载.class文件：  

```
import java.net.URL;
import java.net.URLClassLoader;

public class HelloClassLoader
{
	public static void main( String[] args ) throws Exception
	{
		URL[] urls = {new URL("http://localhost:8000/")};
		URLClassLoader loader = URLClassLoader.newInstance(urls);
		Class c = loader.loadClass("Hello");
		c.newInstance();
	}
}
```

随便写一个HelloWorld：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231201161200.png)

成功加载：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231201162027.png)

成功请求到我们的 /Hello.class 文件，并执行了文件里的字节码，输出了"Hello World"。

所以，作为攻击者，如果我们能够控制目标Java ClassLoader的基础路径为一个http服务器，则可以利
用远程加载的方式执行任意代码了。

# 利用ClassLoader#defineClass直接加载字节码

上一节中我们认识到了如何利用URLClassLoader加载远程class文件，也就是字节码。其实，不管是加载远程class文件，还是本地的class或jar文件，Java都经历的是下面这三个方法调用：  

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231201163100.png)

- loadClass 的作用是从已加载的类缓存、父加载器等位置寻找类（这里实际上是双亲委派机
  制），在前面没有找到的情况下，执行 findClass
- findClass 的作用是根据基础URL指定的方式来加载类的字节码，就像上一节中说到的，可能会在
  本地文件系统、jar包或远程http服务器上读取字节码，然后交给 defineClass
- defineClass 的作用是处理前面传入的字节码，将其处理成真正的Java类  

所以可见，真正核心的部分其实是 defineClass ，他决定了如何将一段字节流转变成一个Java类，Java
默认的 ClassLoader#defineClass 是一个native方法，逻辑在JVM的C语言代码中。

我们可以编写一个简单的代码，来演示如何让系统的 defineClass 来直接加载字节码：  

首先把HelloWorld进行Base64编码：

```java
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.Base64;

public class FileTest {
    public static void main(String[] args) throws Exception {

        String filePath = "xx\\HelloWorld.class";

        // 读取文件内容
        byte[] fileContent = Files.readAllBytes(Paths.get(filePath));

        // 进行Base64编码
        String encodedString = Base64.getEncoder().encodeToString(fileContent);

        System.out.println(encodedString);
    }
}
```

因为我这边编译的java环境和idea java环境不一样报错了，而且Burp还要依赖新的java环境就懒得调了，懂原理即可。

```java
import java.lang.reflect.Method;
import java.util.Base64;

public class FileTest {
    public static void main(String[] args) throws Exception {
        Method defineClass = ClassLoader.class.getDeclaredMethod("defineClass", String.class, byte[].class, int.class, int.class);
        defineClass.setAccessible(true);
        byte[] code = Base64.getDecoder().decode("yv66vgAAAEEAGgoAAgADBwAEDAAFAAYBABBqYXZhL2xhbmcvT2JqZWN0AQAGPGluaXQ+AQADKClWCQAIAAkHAAoMAAsADAEAEGphdmEvbGFuZy9TeXN0ZW0BAANvdXQBABVMamF2YS9pby9QcmludFN0cmVhbTsIAA4BAApIZWxsb1dvcmxkCgAQABEHABIMABMAFAEAE2phdmEvaW8vUHJpbnRTdHJlYW0BAAdwcmludGxuAQAVKExqYXZhL2xhbmcvU3RyaW5nOylWBwAOAQAEQ29kZQEAD0xpbmVOdW1iZXJUYWJsZQEAClNvdXJjZUZpbGUBAA9IZWxsb1dvcmxkLmphdmEAIQAVAAIAAAAAAAEAAQAFAAYAAQAWAAAALQACAAEAAAANKrcAAbIABxINtgAPsQAAAAEAFwAAAA4AAwAAAAIABAADAAwABAABABgAAAACABk=");
        Class hello = (Class) defineClass.invoke(ClassLoader.getSystemClassLoader(),"HelloWorld",code,0,code.length);
        hello.newInstance();
    }
}
```

注意一点，在 defineClass 被调用的时候，类对象是不会被初始化的，只有这个对象显式地调用其构造
函数，初始化代码才能被执行。而且，即使我们将初始化代码放在类的static块中（在本系列文章第一篇中进行过说明），在 defineClass 时也无法被直接调用到。所以，如果我们要使用 defineClass 在目标机器上执行任意代码，需要想办法调用构造函数。

在实际场景中，因为defineClass方法作用域是不开放的，所以攻击者很少能直接利用到它，但它却是我
们常用的一个攻击链 TemplatesImpl 的基石。

# 利用TemplateImpl加载字节码  

虽然大部分上层开发者不会直接使用到defineClass方法，但是Java底层还是有一些类用到了它（否则他
也没存在的价值了对吧），这就是 TemplatesImpl 。  

com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl 这个类中定义了一个内部类
TransletClassLoader ：  

```java
static final class TransletClassLoader extends ClassLoader {
        private final Map<String,Class> _loadedExternalExtensionFunctions;

         TransletClassLoader(ClassLoader parent) {
             super(parent);
            _loadedExternalExtensionFunctions = null;
        }

        TransletClassLoader(ClassLoader parent,Map<String, Class> mapEF) {
            super(parent);
            _loadedExternalExtensionFunctions = mapEF;
        }

        public Class<?> loadClass(String name) throws ClassNotFoundException {
            Class<?> ret = null;
            // The _loadedExternalExtensionFunctions will be empty when the
            // SecurityManager is not set and the FSP is turned off
            if (_loadedExternalExtensionFunctions != null) {
                ret = _loadedExternalExtensionFunctions.get(name);
            }
            if (ret == null) {
                ret = super.loadClass(name);
            }
            return ret;
         }

        /**
         * Access to final protected superclass member from outer class.
         */
        Class defineClass(final byte[] b) {
            return defineClass(null, b, 0, b.length);
        }
    }
```

这个类里重写了 defineClass 方法，并且这里没有显式地声明其定义域。Java中默认情况下，如果一个方法没有显式声明作用域，其作用域为default。所以也就是说这里的 defineClass 由其父类的protected类型变成了一个default类型的方法，可以被类外部调用。  

`com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl`作为[fastjson](https://so.csdn.net/so/search?q=fastjson&spm=1001.2101.3001.7020) <= 1.2.24 反序列化漏洞的其中一条利用链出现过，利用链如下：

```
TemplatesImpl#getOutputProperties()
  TemplatesImpl#newTransformer()
    TemplatesImpl#getTransletInstance()
      TemplatesImpl#defineTransletClasses()
        TransletClassLoader#defineClass()
      Class#newInstance()
```

`TemplatesImpl#getOutputProperties()`和`TemplatesImpl#newTransformer()`都是`public`类型，这里使用`TemplatesImpl#newTransformer()`来实现类加载并初始化。

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;

import java.lang.reflect.Field;
import java.util.Base64;

public class TemplatesImplTest {
    private static void setFiledValue(Object obj, String fieldName, Object fieldValue) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, fieldValue);
    }
    public static void main(String[] args) {
        try {
            byte[] codes = Base64.getDecoder().decode("yv66vgAAADQAIQoABgASCQATABQIABUKABYAFwcAGAcAGQEACXRyYW5zZm9ybQEAcihMY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTtbTGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBAApFeGNlcHRpb25zBwAaAQCmKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABjxpbml0PgEAAygpVgEAClNvdXJjZUZpbGUBABdIZWxsb1RlbXBsYXRlc0ltcGwuamF2YQwADgAPBwAbDAAcAB0BABNIZWxsbyBUZW1wbGF0ZXNJbXBsBwAeDAAfACABABJIZWxsb1RlbXBsYXRlc0ltcGwBAEBjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvcnVudGltZS9BYnN0cmFjdFRyYW5zbGV0AQA5Y29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL1RyYW5zbGV0RXhjZXB0aW9uAQAQamF2YS9sYW5nL1N5c3RlbQEAA291dAEAFUxqYXZhL2lvL1ByaW50U3RyZWFtOwEAE2phdmEvaW8vUHJpbnRTdHJlYW0BAAdwcmludGxuAQAVKExqYXZhL2xhbmcvU3RyaW5nOylWACEABQAGAAAAAAADAAEABwAIAAIACQAAABkAAAADAAAAAbEAAAABAAoAAAAGAAEAAAAIAAsAAAAEAAEADAABAAcADQACAAkAAAAZAAAABAAAAAGxAAAAAQAKAAAABgABAAAACgALAAAABAABAAwAAQAOAA8AAQAJAAAALQACAAEAAAANKrcAAbIAAhIDtgAEsQAAAAEACgAAAA4AAwAAAA0ABAAOAAwADwABABAAAAACABE=");
            byte[][] _bytecodes = new byte[][] {
                    codes,
            };
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

简单过一下这条利用链。

_bytecodes为defineClass的参数：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231203151626.png)

保证_name不为空以进入defineTransletClasses：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231203151317.png)

设置_tfactory的作用：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231203152403.png)

运行结果：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231203152639.png)

另外，值得注意的是， TemplatesImpl 中对加载的字节码是有一定要求的：这个字节码对应的类必须是 com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet 的子类  

所以，需要构造一个特殊的类：

```java
import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;

public class HelloTemplatesImpl extends AbstractTranslet {
	public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {}
    
	public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {}
    
	public HelloTemplatesImpl() {
		super();
		System.out.println("Hello TemplatesImpl");
	}
}
```

它继承了 AbstractTranslet 类，并在构造函数里插入Hello的输出。将其编译成字节码，即可被
TemplatesImpl 执行了：  

在多个Java反序列化利用链，以及fastjson、jackson的漏洞中，都曾出现过 TemplatesImpl 的身影，后面仍然会再次见到它的身影。  

# 利用BCEL ClassLoader加载字节码  

`com.sun.org.apache.bcel.internal.util.ClassLoader`是常常在构造漏洞利用PoC时用到的类。

不过好像在Java 8u251以后就被移除了，以后看到fastjson再说。

# 参考链接

https://wx.zsxq.com/dweb2/index/group/2212251881

https://windysha.github.io/2018/01/18/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3JVM%E4%B9%8BJava%E5%AD%97%E8%8A%82%E7%A0%81%EF%BC%88-class%EF%BC%89%E6%96%87%E4%BB%B6%E8%AF%A6%E8%A7%A3/

https://www.leavesongs.com/PENETRATION/where-is-bcel-classloader.html