---
layout:     post
title:      【Android学习】《Frida协议分析》三
subtitle:   工具学习
date:       2024-02-29
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【Android应用安全实战：Frida协议分析】
    - 【逆向】
    - 【Android】
---

## 第三章

关键代码快速定位的方法有很多，基本思路就是通过Hook一些不变的函数，并打印函数栈信息。不变的函数指的是系统函数和一些没有经过混淆的路径固定的第三方库函数，如okhttp3等。本章介绍的定位思路不止可以应用于Java层函数，对于so层函数也是适用的，如通过Hook libc.so、libdl.so、libart.so、linker等系统函数库来定位。



关键代码快速定位

只要App应用程序想要调用系统函数，不管如何混淆，最终在调用时，系统函数的类名和方法名都是不变的。而在App应用程序的开发中，又不可避免地需要使用系统函数。因此，通过Hook一些系统函数来定位关键代码，是逆向分析中的基本操作。

### 集合的Hook

包括定位散列表HashMap、定位动态数组ArrayList和打印函数堆栈

#### Hook HashMap定位散列表

处理数据常用

![image-20240306194850191](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081406265.png)

#### 打印函数栈

可以使用Log类的getStackTraceString方法

是通过制造异常的方式获取当前函数栈信息

![image-20240306194913851](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081406266.png)

具体如下

![image-20240306195001218](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081406267.png)

在任意想要打印函数栈的地方调用函数showStacks

![image-20240306195021613](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081406268.png)

#### Hook ArrayList定位动态数组

ArrayList在开发中也很常用

![image-20240306195124695](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081406269.png)

### 组件与事件的Hook

####  Toast定位提示

hook这个函数

![image-20240306195419335](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081406270.png)

根据打印的栈信息找到

![image-20240306195511751](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081406271.png)

可以看出App应用程序给的提示信息越多，关键代码越容易被定位，如在Windows逆向中就很喜欢通过对话框来定位关键代码。由此可见，逆向分析的思想是通用的。

#### Hook findViewById定位组件

通过组件id来获取组件，再进行点击事件的绑定或数据的获取等操作

通过SDK中的uiautomatorviewer来查看组件id，发现登录按钮的id为btn_login，接着使用Frida来获取登录按钮id对应的数值，btn_login是R类里面内部类id中的属性。

![image-20240306195610621](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081406272.png)

等于说是hook住了findViewById，当点击btn_login断下来

![image-20240306195641600](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081406273.png)

#### setOnClickListener定位按钮点击事件

一样的原理

![image-20240306200059574](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081406274.png)

### 常用类的Hook

包括定位用户输入、定位JSON数据、定位排序算法、定位字符转化、定位字符串操作和定位Base64编码

#### Hook TextUtils定位用户输入

从EditText组件中获取用户输入的数据后，通常需要判断是否为空。这时可能会使用到TextUtils的isEmpty方法

![image-20240306200229792](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081406275.png)

#### Hook JSONObject定位JSON数据

客户端与服务端进行数据交互时，通常会使用JSON数据作为中间数据进行交互。这时就会用到一些JSON解析相关的类，如JSONObject、Gson等。JSONObject这个类使用的相对较少，因为不是很好用。

![image-20240306200337330](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081406276.png)

上述代码中只Hook了一个重载方法，有需要的可以自行补全。从上述输出结果来看，通过Hook JSONObject类的put方法，定位到的是数据提交的地方。

#### HookCollections定位排序算法

sort

![image-20240306200418271](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081406277.png)

#### Hook String定位字符转换

String类的getBytes方法

![image-20240306200445654](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081406278.png)

#### StringBuilder定位字符串操作

Java中的字符串是只读的，对字符串进行修改、拼接等操作其实都会创建新的字符串来返回。

可以尝试Hook StringBuilder的toString方法来定位App应用程序中的关键字符串。

![image-20240306200513483](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081406279.png)

### 其他类的定位

#### Hook定位接口的实现类

有下面这样的类

![image-20240306200617389](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081406280.png)

实现类只有一个InterfaceDemo

![image-20240306200631265](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081406281.png)

先枚举所有已加载的类，筛选出路径以“com.xiaojianbang”开始的类，使用Java.use通过className获取Frida包装以后的类clazz，再通过Java反射机制提供的方法getInterfaces获取该类实现的所有接口。对这些接口进行遍历，如果其中包含名为“com.xiaojianbang.app.TestAbstract”的接口，则该className就是需要寻找的实现类。

![image-20240306200704478](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081406282.png)

#### Hook定位抽象类的实现类

![image-20240306200738640](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081406283.png)

![image-20240306200744026](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081406284.png)

上述代码先枚举所有已加载的类，筛选出路径以“com.xiaojianbang”开始的类，使用Java.use通过className获取Frida包装以后的类clazz，再通过Java反射机制提供的方法getSuperclass获取其父类resultClass。如果父类名为“com.xiaojianbang.app.TestAbstract”，则该className就是需要寻找的实现类。

![image-20240306200809528](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081406285.png)