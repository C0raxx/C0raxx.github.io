---
layout:     post
title:      【Android学习】《Frida协议分析》二
subtitle:   工具学习
date:       2024-02-28
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【Android应用安全实战：Frida协议分析】
    - 【逆向】
    - 【Android】
---

# 《Android应用安全实战：Frida协议分析》

小结本章通过Frida框架对Android应用的Java层进行了Hook，并通过实战案例进行了巩固，但是对Java层的Hook仅仅是Frida框架中最简单的功能，熟练掌握本章节内容是后续进行逆向实战的基础。

## 第二章

### Frida框架的Hook方法

#### Hook静态方法和实例方法

函数主题需要被包含在Java.perform方法中，参数是一个匿名函数，匿名函数内编写具体的hook代码。分静态和实例

如下源代码

![image-20240306165139065](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081400239.png)

对于静态方法setflag通过implementation直接覆写

对于非静态getinfo类似

![image-20240306165421564](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081400240.png)

自己实验 成功

![image-20240306174139779](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081400241.png)

#### 构造方法

简单来讲就是hook如下这种 构造

![image-20240306174258310](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081400242.png)

只需要使用$init就可以了

![image-20240306175221510](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081400243.png)

自己试 成功

![image-20240306175208770](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081400245.png)

#### Hook重载方法

首先尝试让frida报错来知道有几种重载函数

![image-20240306175623034](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081400246.png)

可以看到不是很好看

![image-20240306175616470](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081400247.png)

在编写正确的hook 要加上overload

![image-20240306175938548](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081400248.png)

#### 所有重载 

感觉复杂了代码流程

![image-20240306180026806](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081400249.png)

#### 对象参数构造

如图传递的参数是money类的money

![image-20240306180330632](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081400250.png)

hook它 打印出来需要调用a的getInfo方法

如果要构造一个money类 则需要用$new出一个类对象

![image-20240306180446325](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081400251.png)

#### 主动调用Java函数

主动调用分静态与示例两种方法

对静态只需要如下

![image-20240306180556168](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081400252.png)

对实例方法的主动调用

```
第一种是创建新对象，第二种是获取已有对象。
```

第一种方法使用$new来创建实例，如主动调用Money类中的getInfo方法，先创建一个Money对象

![image-20240306180711961](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081400253.png)

第二种是获取已有对象 用java.choose 内存中搜索

拥有两个参数，第一个参数是想要找到的类，第二个参数是一个回调函数，包括onMatch和onComplete两个方法，后者是在所有对象搜索完毕后调用，前者是每找到一次就调用一次

![image-20240306180823308](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081400254.png)

### Frida框架Hook类

包括获取和修改类的字段、Hook内部类和匿名类、枚举所有已加载的类、枚举类的所有方法和Hook类的所有方法。

#### 获取和修改类的字段

静态字段只要拿到类就可以访问

实例字段需要得到对象才可以访问，分为创建新对象和获取已有的对象。

源码

![image-20240306193433947](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081400255.png)

需要这样

![image-20240306193505984](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081400256.png)

修改

![image-20240306193524338](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081400257.png)

对于类的实例字段的获取和修改，创建新对象的方法

![image-20240306193547076](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081400258.png)

如果获取已有对象 还是Java.choose方法

![image-20240306193613799](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081400259.png)

#### Hook内部类和匿名类

正确的方法是在类和内部类名之间加上$字符

![image-20240306193711343](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081400260.png)

匿名类的访问。匿名类是一个没有名字的类，是内部类的简化写法，它本质上是继承该类或者实现接口的子类匿名对象。

源码

![image-20240306193736267](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081400261.png)

需要smali来完成类定位

![image-20240306193808314](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081400262.png)

#### 枚举所有已加载的类和枚举类的所有方法

![image-20240306193939251](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081400264.png)

对于方法 需要用到反射

![image-20240306193952569](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081400265.png)

![image-20240306193957086](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081400266.png)

如果还要获取构造放啊 还需额外处理

![image-20240306194019345](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081400267.png)

#### Hook类的所有方法

先把枚举类的所有方法和Hook类的所有重载写出来，用它来Hook测试应用中的Utils类：

![image-20240306194050573](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081400268.png)









