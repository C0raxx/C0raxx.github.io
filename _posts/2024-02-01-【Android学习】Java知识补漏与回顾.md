---
layout:     post
title:      【Android学习】Java知识补漏与回顾
subtitle:   工具学习
date:       2024-02-01
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【CTF】
    - 【逆向】
    - 【Android】
    - 【Java】
---

# Java

最近着重研究安卓逆向碰到了点困难，困难之一在没有系统学习过Java，凭着以前C/C++，和Python的基础硬看，深感不懂Java和正向Android开发流程会遇到的问题，所以年前和寒假尾巴要利用起来顺便把这部分不上。这篇就着重来补漏一些Java方面的缺失，学习来源菜鸟教程和网络课程。

基础知识如数据类型，符号等等就一笔带过，仅作查阅使用。着重看Java的**异常，继承，重写重载，接口等知识**。

## 数据类型

![image-20240202175740653](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250861.png)

## 关于包，类，方法，对象

包是一个类库单元，意思就是包里面有一组类，在同一命名空间被组合到一块，import导入。

类是某些共同特征的实体的集合，抽象的数据类型。简单理解可以理解成把我抽象成人类，然后我只是一个实例化，人类这个类里有行为，参数等等。

方法是类或对象的行为特征的抽象，我就把它理解成函数得了。

## 修饰符

- **default** (即默认，什么也不写）: 在同一包内可见，不使用任何修饰符。使用对象：类、接口、变量、方法。
- **private** : 在同一类内可见。使用对象：变量、方法。 **注意：不能修饰类（外部类）**
- **public** : 对所有类可见。使用对象：类、接口、变量、方法
- **protected** : 对同一包内的类和所有子类可见。使用对象：变量、方法。 **注意：不能修饰类（外部类）**。

![image-20240202183400023](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250863.png)

static 修饰符

- **静态变量：**

	static 关键字用来声明独立于对象的静态变量，无论一个类实例化多少对象，它的静态变量只有一份拷贝。 静态变量也被称为类变量。局部变量不能被声明为 static 变量。

- **静态方法：**

	static 关键字用来声明独立于对象的静态方法。静态方法不能使用类的非静态变量。静态方法从参数列表得到数据，然后计算这些数据。

	

**final 变量：**

final 表示"最后的、最终的"含义，变量一旦赋值后，不能被重新赋值。被 final 修饰的实例变量必须显式指定初始值。



abstract 修饰符

抽象类：抽象类不能用来实例化对象，声明抽象类的唯一目的是为了将来对该类进行扩充。

一个类不能同时被 abstract 和 final 修饰。如果一个类包含抽象方法，那么该类一定要声明为抽象类，否则将出现编译错误。

抽象类可以包含抽象方法和非抽象方法。

这个其实可以用来某些不需要完备类功能，写点方法来用。



synchronized 修饰符

synchronized 关键字声明的方法同一时间只能被一个线程访问。synchronized 修饰符可以应用于四个访问修饰符。



transient 修饰符

序列化的对象包含被 transient 修饰的实例变量时，java 虚拟机(JVM)跳过该特定的变量。

该修饰符包含在定义变量的语句中，用来预处理类和变量的数据类型。

这个其实没太懂用来干嘛，好像是持久化？



volatile 修饰符

volatile 修饰的成员变量在每次被线程访问时，都强制从共享内存中重新读取该成员变量的值。而且，当成员变量发生变化时，会强制线程将变化值回写到共享内存。这样在任何时刻，两个不同的线程总是看到某个成员变量的同一个值。

一个 volatile 对象引用可能是 null。

没看懂+1，后面希望真实场景下可以看到。



## 运算符

![image-20240202184814971](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250864.png)



## 循环结构

![image-20240202185035986](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250865.png)



## character类

和Cpp里面差别不大，主要是一些方法吧。

![image-20240202185217237](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250866.png)





## string类

也熟悉，摆些格式化和方法案例。

格式化两种，printf和format。

format（）返回一个string对象，不是 PrintStream 对象。静态方法。

![image-20240202185532383](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250867.png)

关于方法

https://www.runoob.com/java/java-string.html



## StringBuffer和StringBuilder类

这个是字符串修改的时候需要的，之前还没有好好看过。

这两个类和string类不同在于StringBuffer和StringBuilder类的对象可以被多次修改，不产生新的未使用对象。

```
StringBuilder 类在 Java 5 中被提出，它和 StringBuffer 之间的最大不同在于 StringBuilder 的方法不是线程安全的（不能同步访问）。

由于 StringBuilder 相较于 StringBuffer 有速度优势，所以多数情况下建议使用 StringBuilder 类。
```

来看看示例代码

```
public class RunoobTest{
    public static void main(String args[]){
        StringBuilder sb = new StringBuilder(10);
        sb.append("Runoob..");
        System.out.println(sb);  
        sb.append("!");
        System.out.println(sb); 
        sb.insert(8, "Java");
        System.out.println(sb); 
        sb.delete(5,8);
        System.out.println(sb);  
    }
}
```

上面是一个关于StringBuilder使用。



下面是stringbuffer，似乎是可以保证线程安全。

```
public class Test{
  public static void main(String args[]){
    StringBuffer sBuffer = new StringBuffer("菜鸟教程官网：");
    sBuffer.append("www");
    sBuffer.append(".runoob");
    sBuffer.append(".com");
    System.out.println(sBuffer);  
  }
}
```

至于为什么说是线程安全的，看到一个回答是他的公共方法是同步的，它们在执行期间会获取对象级别的锁，确保在同一时间只有一个线程可以修改一个对象。

https://blog.csdn.net/exodus520/article/details/90415568



## Java Stream,File,IO

是在Java.io包里面的，这个包里几乎包含了所有的操作输入，输出需要的类，所有这些流类都代表着输入源和输出的目标。

其实感觉和C里的流有点类似，不同的是下面的完成方法。

### 从控制台读取多字符输入

这个有`system.in`完成。

`system.in`包进`BufferedReader`对象创建字符流

```
BufferedReader br = new BufferedReader(new 
                      InputStreamReader(System.in));
```

然后通过dowhile读取新的字符，不断调用read()方法。

```
int read( ) throws IOException
```

写进去就是

```
//使用 BufferedReader 在控制台读取字符
 
import java.io.*;
 
public class BRRead {
    public static void main(String[] args) throws IOException {
        char c;
        // 使用 System.in 创建 BufferedReader
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        System.out.println("输入字符, 按下 'q' 键退出。");
        // 读取字符
        do {
            c = (char) br.read();
            System.out.println(c);
        } while (c != 'q');
    }
}
```

重点是do while里面的调用br的read()方法。

### 控制台输出

相对简单，继承关系是*PrintStream 继承了 OutputStream类，并且实现了方法 write()。这样，write() 也可以用来往控制台写操作。*

`void write(int byteval)`

### 读写文件

有关的事FileInputStream和FileOutputStream

![image-20240202213605435](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250868.png)

#### FileInStream

文件读取数据

```
InputStream f = new FileInputStream("C:/java/hello");
```

![image-20240202213735739](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250869.png)

还有俩

![image-20240202213742111](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250870.png)

#### FileOutputStream

输出到哪里去，和上面相对的。

![image-20240202213835703](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250871.png)



还有关于目录的操作，就不写了，在下面url。

https://www.runoob.com/java/java-files-io.html



## Scanner类

也是获取用户输入的。

```
Scanner s = new Scanner(System.in);
```

一般用next()和nextLine()获取下一个，然后用hasNext和hasNextLine来判断是否还有输入。

```
import java.util.Scanner; 
 
public class ScannerDemo {
    public static void main(String[] args) {
        Scanner scan = new Scanner(System.in);
        // 从键盘接收数据
 
        // next方式接收字符串
        System.out.println("next方式接收：");
        // 判断是否还有输入
        if (scan.hasNext()) {
            String str1 = scan.next();
            System.out.println("输入的数据为：" + str1);
        }
        scan.close();
    }
}
```

next()和nextLine() 区别

- 一定要读取到有效字符后才可以结束输入。
- 对输入有效字符之前遇到的空白，next() 方法会自动将其去掉。
- 只有输入有效字符后才将其后面输入的空白作为分隔符或者结束符。 next() 不能得到带有空格的字符串。

o

- 以Enter为结束符,也就是说 nextLine()方法返回的是输入回车之前的所有字符。 
- 2、可以获得空白。

## 异常处理

三种类型的异常。

![image-20240202214912708](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250872.png)



### 异常类的层次

都是从java.lang.Exception继承下来的子类。

然Exception是Throwable类的子类，他还有一个error子类，而Java通常不捕获错误。

而Exception有两个主要子类

![image-20240202215115293](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250873.png)



### 内置异常类

如下是非检查性的异常

![image-20240202215149777](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250874.png)

关于检查性异常

![image-20240202215228170](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250875.png)

### 异常方法

主要是Throwable

![image-20240202215302566](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250876.png)

### 关于捕获异常

比较熟悉的try catch

![image-20240202215349644](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250877.png)



### 关于throws/throw关键字

throw我熟悉一点

![image-20240202215703817](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250878.png)



关于throws，菜鸟教程上没怎么看懂。

是一种异常处理的方式，和throw不一样的是他不是扔出一个异常，而是可能出现的异常没然后它把异常往调用该方法者上面扔，指导JVM。

说实话，不太明白throws存在的意义，问gpt给一段实例。

```
public class Example {
    public static void main(String[] args) {
        try {
            // 调用可能抛出异常的方法
            processInput();
        } catch (IOException e) {
            // 捕获并处理IOException异常
            System.out.println("发生了IO异常：" + e.getMessage());
        }
    }

    public static void processInput() throws IOException {
        // 这里假设有可能发生IOException异常
        throw new IOException("文件读取错误");
    }
}

```

我觉得可以直接如下

```
public class Example {
    public static void main(String[] args) {
        try {
            throw new IOException("文件读取错误");
        } catch (IOException e) {
            // 捕获并处理IOException异常
            System.out.println("发生了IO异常：" + e.getMessage());
        }
    }
}
```

gpt的解释

![image-20240202220937934](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250879.png)

好吧。。

### Finally

异常后面的代码块，不管如何都会执行，做善后性质的语句。

### try-with-resource

新增的语法糖。。感觉就是加了一个try附加条件

![image-20240202221701076](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250880.png)



## 继承

是很重要的一块了，就是子类继承父类的特征和行为，以此来避免重复代码编写和维护性差的问题，也是Cpp里使用频率相当高的概念。

java这么写

![image-20240202222200189](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250881.png)

 不支持多继承，但可以多重继承

![image-20240202222319916](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250882.png)



虽说如此，但是多继承的特性可以改用`implements`的继承多个接口来曲线救国。

```
public interface A {
    public void eat();
    public void sleep();
}
 
public interface B {
    public void show();
}
 
public class C implements A,B {
}
```



super 父类

this自己的引用



### final

用来修饰变量，方法或者类。

这玩意就是最终的了，不可被继承，不可被重写。



### 构造器

如果父类的构造器带有参数，则必须在子类的构造器中显式地通过 super 关键字调用父类的构造器并配以适当的参数列表。

如果父类构造器没有参数，则在子类的构造器中不需要使用 super 关键字调用父类构造器，系统会自动调用父类的无参构造器。

示例代码

```
class SuperClass {
  private int n;
  SuperClass(){
    System.out.println("SuperClass()");
  }
  SuperClass(int n) {
    System.out.println("SuperClass(int n)");
    this.n = n;
  }
}
// SubClass 类继承
class SubClass extends SuperClass{
  private int n;
  
  SubClass(){ // 自动调用父类的无参数构造器
    System.out.println("SubClass");
  }  
  
  public SubClass(int n){ 
    super(300);  // 调用父类中带有参数的构造器
    System.out.println("SubClass(int n):"+n);
    this.n = n;
  }
}
// SubClass2 类继承
class SubClass2 extends SuperClass{
  private int n;
  
  SubClass2(){
    super(300);  // 调用父类中带有参数的构造器
    System.out.println("SubClass2");
  }  
  
  public SubClass2(int n){ // 自动调用父类的无参数构造器
    System.out.println("SubClass2(int n):"+n);
    this.n = n;
  }
}
public class TestSuperSub{
  public static void main (String args[]){
    System.out.println("------SubClass 类继承------");
    SubClass sc1 = new SubClass();
    SubClass sc2 = new SubClass(100); 
    System.out.println("------SubClass2 类继承------");
    SubClass2 sc3 = new SubClass2();
    SubClass2 sc4 = new SubClass2(200); 
  }
}
```

## 重写

简单来讲就是对父类原本有的某些方法进行重新编写，但是**返回值和形参都不能改变**。

![image-20240202222934098](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250883.png)

输出

![image-20240202222942018](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250884.png)

可以看到虽然是b是animal 但是是调用了dog的move方法。

重写规则，其实就是返回值和形参最重要

- 参数列表与被重写方法的参数列表必须完全相同。
- 返回类型与被重写方法的返回类型可以不相同，但是必须是父类返回值的派生类（java5 及更早版本返回类型要一样，java7 及更高版本可以不同）。
- 访问权限不能比父类中被重写的方法的访问权限更低。例如：如果父类的一个方法被声明为 public，那么在子类中重写该方法就不能声明为 protected。
- 父类的成员方法只能被它的子类重写。
- 声明为 final 的方法不能被重写。
- 声明为 static 的方法不能被重写，但是能够被再次声明。
- 子类和父类在同一个包中，那么子类可以重写父类所有方法，除了声明为 private 和 final 的方法。
- 子类和父类不在同一个包中，那么子类只能够重写父类的声明为 public 和 protected 的非 final 方法。
- 重写的方法能够抛出任何非强制异常，无论被重写的方法是否抛出异常。但是，重写的方法不能抛出新的强制性异常，或者比被重写方法声明的更广泛的强制性异常，反之则可以。
- 构造方法不能被重写。
- 如果不能继承一个类，则不能重写该类的方法。



如果需要在重写方法中调用父类的被重写方法，写super.xxx



## 重载

是指在一个类里面，方法相同，但**参数不同**，返回值可以相同也可以不同。



- 被重载的方法必须改变参数列表(参数个数或类型不一样)；
- 被重载的方法可以改变返回类型；
- 被重载的方法可以改变访问修饰符；
- 被重载的方法可以声明新的或更广的检查异常；
- 方法能够在同一个类中或者在一个子类中被重载。
- 无法以返回值类型作为重载函数的区分标准。



示例代码

```
public class Overloading {
    public int test(){
        System.out.println("test1");
        return 1;
    }
 
    public void test(int a){
        System.out.println("test2");
    }   
 
    //以下两个参数类型顺序不同
    public String test(int a,String s){
        System.out.println("test3");
        return "returntest3";
    }   
 
    public String test(String s,int a){
        System.out.println("test4");
        return "returntest4";
    }   
 
    public static void main(String[] args){
        Overloading o = new Overloading();
        System.out.println(o.test());
        o.test(1);
        System.out.println(o.test(1,"test3"));
        System.out.println(o.test("test4",1));
    }
}
```

![image-20240202223249192](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250885.png)

下面是两者的总结，菜鸟教程写的很好，不乱bb了。

![image-20240202223315746](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250886.png)

## 多态

这里写的比较笼统。

文章主要用的是重写的方法实现。

文中提到了虚函数，和C++里面的差不太多。

https://www.runoob.com/java/java-polymorphism.html

算是归纳吧，主要是重写，接口和抽象。



## 抽象类

这玩意之前有写过，类似个功能不健全的类，无法实例化对象吗，但是诸如方法等等的访问方式和普通类一样。

抽象类必须被继承才可以使用，他表现得是一种继承关系，一个类只能继承一个抽象类，但是一个类却可以实现很多接口。

现在写一个抽象类

```
/* 文件名 : Employee.java */
public abstract class Employee
{
   private String name;
   private String address;
   private int number;
   public Employee(String name, String address, int number)
   {
      System.out.println("Constructing an Employee");
      this.name = name;
      this.address = address;
      this.number = number;
   }
   public double computePay()
   {
     System.out.println("Inside Employee computePay");
     return 0.0;
   }
   public void mailCheck()
   {
      System.out.println("Mailing a check to " + this.name
       + " " + this.address);
   }
   public String toString()
   {
      return name + " " + address + " " + number;
   }
   public String getName()
   {
      return name;
   }
   public String getAddress()
   {
      return address;
   }
   public void setAddress(String newAddress)
   {
      address = newAddress;
   }
   public int getNumber()
   {
     return number;
   }
}
```

如下实例化是错误的

![image-20240202224036602](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250887.png)

要继承

```
/* 文件名 : Salary.java */
public class Salary extends Employee
{
   private double salary; //Annual salary
   public Salary(String name, String address, int number, double
      salary)
   {
       super(name, address, number);
       setSalary(salary);
   }
   public void mailCheck()
   {
       System.out.println("Within mailCheck of Salary class ");
       System.out.println("Mailing check to " + getName()
       + " with salary " + salary);
   }
   public double getSalary()
   {
       return salary;
   }
   public void setSalary(double newSalary)
   {
       if(newSalary >= 0.0)
       {
          salary = newSalary;
       }
   }
   public double computePay()
   {
      System.out.println("Computing salary pay for " + getName());
      return salary/52;
   }
}
```

里面的Salary从Employee继承了七个成员方法，且获取三个成员变量。

```
/* 文件名 : AbstractDemo.java */
public class AbstractDemo
{
   public static void main(String [] args)
   {
      Salary s = new Salary("Mohd Mohtashim", "Ambehta, UP", 3, 3600.00);
      Employee e = new Salary("John Adams", "Boston, MA", 2, 2400.00);
 
      System.out.println("Call mailCheck using Salary reference --");
      s.mailCheck();
 
      System.out.println("\n Call mailCheck using Employee reference--");
      e.mailCheck();
    }
}
```

输出就如下

![image-20240202224147319](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250888.png)

也算是不错的写法。



### 抽象方法

*如果你想设计这样一个类，**该类包含一个特别的成员方法，该方法的具体实现由它的子类确定**，那么你可以在父类中声明该方法为抽象方法。*

说实话，真的很抽象。。。。

![image-20240202224320451](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250889.png)

总结，来解决我的困惑

![image-20240202224348152](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250890.png)



## 封装

这里也是总结性质

实现封装

```
/* 文件名: EncapTest.java */
public class EncapTest{
 
   private String name;
   private String idNum;
   private int age;
 
   public int getAge(){
      return age;
   }
 
   public String getName(){
      return name;
   }
 
   public String getIdNum(){
      return idNum;
   }
 
   public void setAge( int newAge){
      age = newAge;
   }
 
   public void setName(String newName){
      name = newName;
   }
 
   public void setIdNum( String newId){
      idNum = newId;
   }
}
```

## 接口

是JAVA的一个抽象类型，抽象方法的集合，通常用`interface`来声明。

一个类通过继承接口的方式，从而来**继承接口的抽象方法**。

### 接口与类相似点

接口也可以有很多方法，被存在`.java`里面，文件名就是接口名。

字节码在`.class`里面

### 接口与类区别

但是接口不可以用于实例化，也没有构造方法。

里面的所有方法必须是抽象方法。

接口不能包含成员变量，除了 static 和 final 变量。

接口不是被类继承了，而是要被类实现，接口支持多继承。

### 接口特性

- 接口中每一个方法也是隐式抽象的,接口中的方法会被隐式的指定为 **public abstract**（只能是  public abstract，其他修饰符都会报错）。
- 接口中可以含有变量，但是接口中的变量会被隐式的指定为 **public static final** 变量（并且只能是 public，用 private 修饰会报编译错误）。
- 接口中的方法是不能在接口中实现的，只能由实现接口的类来实现接口中的方法。



### 抽象类和接口的区别

![image-20240202224745263](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250891.png)

### 接口声明

```
[可见度] interface 接口名称 [extends 其他的接口名] {
        // 声明变量
        // 抽象方法
}

例子

/* 文件名 : NameOfInterface.java */
import java.lang.*;
//引入包
 
public interface NameOfInterface
{
   //任何类型 final, static 字段
   //抽象方法
}
```



### 接口的实现

类要实现接口所有的方法，否则类必须声明成抽象类。

类使用implements关键字实现接口。在类声明中，Implements关键字放在class声明后面。

```
...implements 接口名称[, 其他接口名称, 其他接口名称..., ...] ...

例子

/* 文件名 : MammalInt.java */
public class MammalInt implements Animal{
 
   public void eat(){
      System.out.println("Mammal eats");
   }
 
   public void travel(){
      System.out.println("Mammal travels");
   } 
 
   public int noOfLegs(){
      return 0;
   }
 
   public static void main(String args[]){
      MammalInt m = new MammalInt();
      m.eat();
      m.travel();
   }
}
```

结果

![image-20240202224934951](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202402022250892.png)

### 接口的继承



接口也可以继承的

```
// 文件名: Sports.java
public interface Sports
{
   public void setHomeTeam(String name);
   public void setVisitingTeam(String name);
}
 
// 文件名: Football.java
public interface Football extends Sports
{
   public void homeTeamScored(int points);
   public void visitingTeamScored(int points);
   public void endOfQuarter(int quarter);
}
 
// 文件名: Hockey.java
public interface Hockey extends Sports
{
   public void homeGoalScored();
   public void visitingGoalScored();
   public void endOfPeriod(int period);
   public void overtimePeriod(int ot);
}
```









