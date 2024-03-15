---
layout:     post
title:      【Android学习】《Android进阶解密》三之四
subtitle:   工具学习
date:       2024-03-14
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【Android进阶解密】
    - 【逆向】
    - 【Android】
---

# 消息循环创建

这里跟三之三一样，同样是作为三之二的补充

消息循环创建的时机在这

![image-20240315111651122](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151124628.png)

我们看看对应的代码

看看当时的分析

```
可以看到在注释1处通过反射获得了android.app.ActivityThread类，接下来在注释2处获得了ActivityThread的main方法，并将main方法传入注释3处的Zygote中的MethodAndArgsCaller类的构造方法中。

在注释3处抛出的MethodAndArgsCaller异常会被Zygote的main方法捕获
```

![image-20240315111812943](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151124629.png)

我们现在要关注的是在这行代码里面

![image-20240315111903831](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151124630.png)

会抛出一个MethodAndArgsCaller异常，这个异常会被ZygoteInit的main方法捕获

看看这个main

喏 main方法 里面run 很熟悉

![image-20240315111925736](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151124631.png)

继续进入MethodAndArgsCaller往下看run

![image-20240315112006137](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151124632.png)

根据之前的分析啊 这个invoke的method指的是ActivityThread的main方法，margs指的是应用程序进程的启动参数，那我们的目光就投向ActivityThread的main方法

![image-20240315112155970](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151124633.png)

大概到这里就差不多了

在注释1处创建主线程的消息循环Looper，

在注释2处创建ActivityThread。

在注释3处判断Handler类型的sMainThreadHandler 是否为null，

如果为null则在注释4处获取H类并赋值给sMainThreadHandler，这个H类继承自Handler，是ActivityThread的内部类，**用于处理主线程的消息循环，在第4章、第5章我们将会经常提到它。**

在注释5处**调用Looper的loop方法，使得Looper开始处理消息**。

结束

# 小结

到这里第三章也就结束了，其实主要是三之二里面的内容，后面两小章（关于binder线程池创建和消息循环创建）其实都是关于应用程序进程启动当中的一些查缺补漏。

埋个坑 就是上面的**H类**