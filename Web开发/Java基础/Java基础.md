# Java基础

## 一、注释

1. 单行注释：//注释

2. 多行注释：/* 注释 */

3. 文档注释：/** + 回车

   ![image-20230619105503992](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/202306191055079.png)

## 二、标识符

- 所有的标识符都应该以字母、美元符或者下划线开始。

- **不能使用关键字作为变量名或方法名**
- 标识符是**大小写敏感**的

非法标识符举例： 123abc、-salary、#abc

## 三、数据类型

Java是**强类型语言**，所有变量都必须先定义后才能使用。Java的数据类型分为两大类：基本类型（primitive type）和引用类型（reference type）。![](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/image-20230530151041689.png)

1. String是类，不是关键字。
2. 二进制0b，十六进制0x，八进制0。

## 四、类型转换

![image-20230530163205719](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/202305301632743.png)

## 五、变量

### 1. 变量作用域

![image-20230601165527562](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/202306011655626.png)

![image-20230601170409250](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/202306011704297.png)

![image-20230601170511394](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/202306011705440.png)

![image-20230601170727412](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/202306011707473.png)

## 六、Scanner对象

![image-20230619113334493](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/202306191133532.png)

![image-20230619181250212](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/202306191812257.png)

![image-20230619182607873](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/202306191826894.png)

## 七、Java方法

![image-20230807160312490](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/202308071603743.png)

## 八、方法的重载

![image-20230807165146914](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/202308071651955.png)

## 九、命令行传参

简单复习一下命令行如何编译执行，原理先不管。

javac编译  java执行

## 十、可变参数

![image-20230809152031187](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/202308091520258.png)

![image-20230809153721872](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/202308091537911.png)

## 十一、Arrays类

![image-20230813141410375](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/202308131414451.png)

## 十二、面向对象

面向对象编程的本质就是：以类的方式组织代码，以对象的组织（封装数据）。

三大特性：**封装、继承、多态**

类是一种抽象的数据类型，它是对某一类事务整体描述/定义，到那时并不能代表某一个具体的事务。如动物、植物......

对象是抽象概念的具体实例：张三是人的一个具体实例。

![image-20230814155644692](https://picgo-1300397932.cos.ap-nanjing.myqcloud.com/picgo/202308141556789.png)

## 十三、封装

属性私有，get/set

alt + insert快速生成

## 十四、继承

注：ctrl+h查看数据结构

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20230917131050.png)

## 十五、重写

非静态、public才能重写（子类对父类的方法

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20230917142821.png)

## 十六、多态

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20230917143412.png)

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20230917145720.png)

如果子类重写了父类的方法，那么就执行子类的

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20230917145827.png)

## 十六、instanceof和类型转换

instanceof：判断其左边对象是否为其右边类的实例

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20230917152654.png)

上图还可以写成((Student) obj).go()

## 十七、抽象类

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20230917155136.png)

## 十八、接口

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20230917155702.png)

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20230917160719.png)

## 十九、内部类

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20230917161225.png)

## 二十、异常

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20230918091124.png)

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20230918091628.png)

catch(想捕获的异常类型，见上图，最高为Throwable)

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20230918092237.png)

## 二十一、集合框架



## 二十二、其他

常用类

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20230918093759.png)

![image-20230918093845708](C:\Users\86158\AppData\Roaming\Typora\typora-user-images\image-20230918093845708.png)

![image-20230918093914867](C:\Users\86158\AppData\Roaming\Typora\typora-user-images\image-20230918093914867.png)

![](https://cdn.jsdelivr.net/gh/ITroyeSivan/picture/blogpictures/20230918094008.png)

太多了：https://www.bilibili.com/video/BV12J41137hu/?p=80
