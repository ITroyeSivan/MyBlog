# 类加载

 ## 一、类加载过程

程序员编写的Java源程序（.java文件）在经过编译器编译之后被转换成字节代码（.class 文件），类加载器将.class文件中的二进制数据读入到内存中，将其放在方法区内，然后在堆区创建一个java.lang.Class对象，用来封装类在方法区内的数据结构。

类加载的最终产品是位于堆区中的Class对象，Class对象封装了类在方法区内的数据结构，并且向Java程序员提供了访问方法区内的数据结构的接口。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231110163237.png)



## 二、类生命周期

类的生命周期包括：加载、验证、准备、解析、初始化、使用、卸载7个阶段。其中加载、验证、准备、初始化、卸载5个阶段是按照这种顺序按部就班的开始，而解析阶段则不一定：某些情况下，可以在初始化之后再开始，这是为了支持Java语言的运行时绑定（也称为动态绑定或晚期绑定，其实就是多态），例如子类重写父类方法。

![image-20231112102603830](C:\Users\86158\AppData\Roaming\Typora\typora-user-images\image-20231112102603830.png)

注意：这里写的是按部就班的开始，而不是按部就班地进行或完成，因为这些阶段通常都是互相交叉混合式进行的，通常会在一个阶段执行过程中调用、激活另外一个阶段。

### 1、加载

加载阶段会做3件事情：

- 通过一个类的全限定名来获取定义此类的二进制字节流。
- 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。
- Java堆中生成一个代表这个类的java.lang.Class对象，作为对方法区中这些数据的访问入口。

此处第一点并没指明要从哪里获取、怎样获取，因此这里给开发人员预留了扩展空间。许多Java技术就建立在此基础上，例如：

- 从ZIP包读取，如JAR、WAR。
- 从网络中获取，这种场景最典型应用场景应用就是Applet。
- 运行时计算生成，使用较多场景是动态代理技术，如spring AOP。

加载阶段完成后，虚拟机外部的二进制字节流就按照虚拟机所需的格式存储在方法区之中，而且在Java堆中也创建一个java.lang.Class类的对象，这样便可以通过该对象访问方法区中的这些数据。

### 2、验证

确保被加载类的正确性，分为4个验证阶段

- 文件格式验证
- 元数据验证
- 字节码验证
- 符号引用验证

验证阶段非常重要的，但不是必须的，它对程序运行期没有影响，如果所引用的类经过反复验证，那么可以考虑采用-Xverifynone参数来关闭大部分的类验证措施，以缩短虚拟机类加载的时间。

### 3、准备

为类的静态变量分配内存，并初始化默认值，这些内存是在方法区中分配，需要注意以下几点：

• 此处内存分配的变量仅包含类变量（static），而不包括实例变量，实例变量会随着对象实例化被分配在java堆中。

• 这里默认值是数据类型的默认值（如0、0L、null、false），而不是代码中被显示的赋予的值。

• 如果类字段的字段属性表中存在ConstatntValue属性，即同时被final和static修饰，那么在准备阶段变量value就会被初始化为ConstValue属性所指定的值。

### 4、解析

解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程，解析动作主要针对类或接口、字段、类方法、接口方法、方法类型、方法句柄和调用点限定符7类符号引用进行。符号引用就是一组符号来描述目标，可以是任何字面量。

直接引用就是直接指向目标的指针、相对偏移量或一个间接定位到目标的句柄。

### 5、初始化

为类的静态变量赋予正确的初始值，JVM负责对类进行初始化，主要对类变量进行初始化。初始化阶段是执行类构造器`<clinit>()`方法的过程。

`<clinit>()`方法是由编译器自动收集类中的所有类变量赋值动作和静态语句static{}块中的语句合并产生的，编译器收集的顺序是由语句在源文件出现的顺序所决定的。静态语句块中只能访问到定义在静态语句块之前的变量，定义在之后的变量可以赋值，但不能访问。如下所示：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231112111326.png)

`<clinit>()`方法与类构造函数不一样，不需要显示调用父类构造函数，虚拟机会保证在子类的`<clinit>()`方法执行之前，父类的`<clinit>()`方法已执行完毕。

由于父类的`<clinit>()`方法首先执行，意味着父类中的静态语句块要优先于子类的变量赋值操作，如下所示，最终得出的值是2，而不是1。

```java
public class TestClassLoader {
    public static int A = 1;
    static {
        A = 2;
//        System.out.println(A);
    }
    
    static class Sub extends TestClassLoader {
        public static int B = A;
    }
    
    public static void main(String[] args) {
        System.out.println(Sub.B);
    }
}
```

`<clinit>()`方法对于类和接口来说，并不是必须的，若类没有静态语句块，也没有对变量赋值操作，则不会生成`<clinit>()`方法。

接口与类不同的是，接口不需要先执行父类的`<clinit>()`方法，只有父接口定义的变量使用时，父接口才会被初始化。另外接口的实现类也不会先执行接口的`<clinit>()`方法。

虚拟机保证当多线程去初始化类时，只会有一个线程去执行`<clinit>()`方法，而其他线程则被阻塞。

`<clinit>()`方法和`<init>()`方法区别：

执行时机不同：init方法是对象构造器方法，在new一个对象并调用该对象的constructor方法时才会执行。clinit方法是类构造器方法，是在JVM加载期间的初始化阶段才会调用。

执行目的不同：init是对非静态变量解析初始化，而clinit是对静态变量，静态代码块进行初始化。

### 注：类的初始化

类实例化和初始化概念

**类实例化**：是指创建一个类的实例对象的过程，由类创建的对象，在构造一个实例时，需要在内存中开辟空间，即new Object()。 

**类初始化**：是指类中各个类成员（被static修饰的成员变量）赋初始值的过程，是类生命周期中的一个阶段；即实例化基础上对对象赋初始值。

**类初始化的时机**

有且只有5中情况下必须立即对类进行初始化：

- 遇到**new、getstatic、putstaic、invokestatic** 四条字节码指令，如果没有初始化则需要先进行初始化。

注：数组类型初始化只会初始化数组类型本身，不会初始化相关类型，例如：new String[]，此时只会初始化String[]即Ljava.lang.String，但是不会触发初始化String。

**常见场景：**

1、使用new关键字实例化对象。

2、读取或者设置一个类的静态字段（被final修饰，已在编译器把结果放在常量池的静态字段除外）的时候。

3、调用一个类的静态方法时。

- 使用java.lang.reflect进行反射调用，如果未初始化，触发初始化
- 初始化一个类时，若父类未初始化，则先触发父类初始化
- 虚拟机启动时，用户需要指定一个执行main方法的主类，虚拟机会先初始化这个类
- 如果一个java.lang.invoke.MethodHandle 实例最后的解析结果的REF-getstatic、REF-pubstatic、REF-invokestatic方法句柄，并且这个方法句柄所在对应的类没有进行初始化，则需要先触发其初始化

以上初始化称为主动引用，除此之外所有引用类的方式都不会触发初始，称为**被动引用**。

**被动引用案例**

- 通过子类引用父类的静态字段，不会导致子类的初始化。
- 通过数组定义来引用类，不会触发此类的初始化。
- 常量在编译阶段会存入调用类的常量池中，本质上并没有直接引用到定义常量的类，因此不会触发定义常量的初始化；例如：调用public static final String CONSTANT="XXX"。

**Java对象创建的几种方式**

（new、反射、克隆 ）

- new Object()，以new方式调用构造函数创建对象；
- 使用class类的newInstance()方法（反射）；
- 使用Constructor类的newInstance()方法（反射）；
- 使用clone()方法创建对象（克隆）；
- 使用序列化|反序列化机制创造对象（深克隆）；

**Java对象创建的过程**

1. 在堆内存中开辟一块空间；

2. 给开辟的空间分配一个地址；

3. 对所有非静态成员加载到所开辟的空间；

4. 对非静态成员变量进行默认值初始化；

5. 调用构造函数；

6. 构造函数入栈执行时，先隐式三步（super()、初始化、构造代码块），再执行构造函数代码；

   > 构造函数和构造器的执行顺序：
   >
   > 1. 父类的类构造器<clinit>()
   > 2. 子类的类构造器<clinit>()
   > 3. 父类的成员变量和实例代码块
   > 4. 父类的构造函数
   > 5. 子类的成员变量和实例代码块
   > 6. 子类的构造函数

7. 构造函数执行完弹出栈后，把空间分配的地址给引用对象；

核心步骤：**检查类释放被加载——为新生对象分配内存——初始化设定零值——必要的设置——执行<init>方法。**



 **类构造器`<clinit>()`与实例构造器`<init>()`区别**：类构造器<clinit>()与实例构造器<init>()不同，它不需要显式调用，虚拟机自己保证子类<clinit>()之前 调用父类<clinit>()。在同一个类加载器下，一个类只会被初始化一次，但可以任意实例化对象；即：一个类生命周期中，类构造器最多会被虚拟机调用一次，而实例化构造器会被调用多次，只要还在创建对象。

**Java对象实例初始化**

涉及三种执行对象的初始化：

- **实例变量初始化**：实例变量直接赋值。
- **实例代码块初始化**：实例代码块赋值，编译器会将其中的代码放在类的构造函数中去，并且这些代码会被放在超类构造函数调用语句之后，构造函数本身代码之前；这也是为什么Java要求构造函数第一句必须是super();即超类的类构造函数的调用语句；Java要求实例化之前必须实例化其父类，以保证完整性。
- **构造函数初始化**：

```java
public class A{
    private int i=2;//实例变量初始化
    
    {
        i++;//实例代码块初始化
    }
    
    public A(){}//构造函数初始化
}
```

# 类加载器简介

**引导类加载器(Bootstrap ClassLoader):**

这个类加载器负责将\lib目录下的类库加载到虚拟机内存中,用来加载java的核心库,此类加载器并不继承于java.lang.ClassLoader,不能被java程序直接调用,代码是使用C++编写的.是虚拟机自身的一部分.

**扩展类加载器(Extendsion ClassLoader):**

这个类加载器负责加载\lib\ext目录下的类库,用来加载java的扩展库,开发者可以直接使用这个类加载器.

**App类加载器(Application ClassLoader):**

这个类加载器负责加载用户类路径(CLASSPATH)下的类库,一般我们编写的java类都是由这个类加载器加载,这个类加载器是CLassLoader中的getSystemClassLoader()方法的返回值,所以也称为系统类加载器.一般情况下这就是系统默认的类加载器.

除此之外,我们还可以加入自己定义的类加载器,以满足特殊的需求,需要继承java.lang.ClassLoader类.

类加载器之间的层次关系如下图:

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231113124537.png)

# 双亲委派模型

双亲委派的意思是如果一个类加载器需要加载类，那么首先它会把这个类请求委派给父类加载器去完成，每一层都是如此。一直递归到顶层，当父加载器无法完成这个请求时，子类才会尝试去加载。这里的双亲其实就指的是父类，没有mother。父类也不是我们平日所说的那种继承关系，只是调用逻辑是这样。

使用双亲委派模型的好处在于**Java类随着它的类加载器一起具备了一种带有优先级的层次关系**。例如类java.lang.Object，它存在在rt.jar中，无论哪一个类加载器要加载这个类，最终都是委派给处于模型最顶端的Bootstrap ClassLoader进行加载，因此Object类在程序的各种类加载器环境中都是同一个类。相反，如果没有双亲委派模型而是由各个类加载器自行加载的话，如果用户编写了一个java.lang.Object的同名类并放在ClassPath中，那系统中将会出现多个不同的Object类，程序将混乱。因此，如果开发者尝试编写一个与rt.jar类库中重名的Java类，可以正常编译，但是永远无法被加载运行。

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231114124032.png)

看下jdk1.8源码java.lang.ClassLoader.loadClass()方法实现：

```java
public Class<?> loadClass(String name) throws ClassNotFoundException {
        return loadClass(name, false);
    }

    protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```

上面代码注释写的很清楚，首先调用findLoadedClass方法检查是否已加载过这个类，如果没有就调用parent的loadClass方法，从底层一级级往上。如果所有ClassLoader都没有加载过这个类，就调用findClass方法查找这个类，然后又从顶层逐级向下调用findClass方法，最终都没找到就抛出ClassNotFoundException。这样设计的目的是保证安全性，防止系统类被伪造。

为了便于理解，以下是加载逻辑示意图：

![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/20231114130557.png)

# 自定义类加载器

若要实现自定义类加载器，只需要继承java.lang.ClassLoader 类，并且重写其findClass()方法即可。java.lang.ClassLoader 类的基本职责就是根据一个指定的类的名称，找到或者生成其对应的字节代码，然后从这些字节代码中定义出一个 Java 类，即 java.lang.Class 类的一个实例。除此之外，ClassLoader 还负责加载 Java 应用所需的资源，如图像文件和配置文件等，ClassLoader 中与加载类相关的方法如下：

方法说明

getParent() 返回该类加载器的父类加载器。

loadClass(String name) 加载名称为 二进制名称为name 的类，返回的结果是 java.lang.Class 类的实例。

findClass(String name) 查找名称为 name 的类，返回的结果是 java.lang.Class 类的实例。

findLoadedClass(String name) 查找名称为 name 的已经被加载过的类，返回的结果是 java.lang.Class 类的实例。

resolveClass(Class<?> c) 链接指定的 Java 类。

简单了解，不展开。





















# 参考链接

1. https://github.com/Y4tacker/JavaSec/blob/main/1.%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/ClassLoader(%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6)/ClassLoader(%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6).md
2. https://drun1baby.top/2022/06/03/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%9F%BA%E7%A1%80%E7%AF%87-05-%E7%B1%BB%E7%9A%84%E5%8A%A8%E6%80%81%E5%8A%A0%E8%BD%BD/
3. https://segmentfault.com/a/1190000023876273
4. https://zhuanlan.zhihu.com/p/648512847