---
layout:     post
title:      【Android学习】《Android进阶解密》三之二
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

## 应用程序进程启动过程介绍

内容开始多起来了
应用程序进程创建过程因为内容比较多，分为两部分来讲，一部分是**AMS发送启动应用程序进程请求**，以及**Zygote接收请求并创建应用程序进程**。

## AMS发送启动应用程序进程请求

这是第一个部分哈，时序图先给出

![image-20240314091132806](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151029562.png)

AMS想要启动应用程序进程，AMS会通过调用startProcessLocked方法向Zygote进程发送请求。

我在里面找了两个重载

![image-20240314091420452](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151029563.png)

主要是关注第二个吧

![image-20240314092335690](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151029564.png)

![image-20240314092347371](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151029565.png)

在注释1处得到创建应用程序进程的用户ID

在注释2处对用户组ID（gids）进行创建和赋值

在注释3处如果entryPoint为null，则赋值为android.app.ActivityThread，这个值就是应用程序进程主线程的类名

在注释4处调用Process的start方法，将此前得到的应用程序进程用户ID和用户组ID传进去，第一个参数entryPoint我们得知是android.app.ActivityThread



前三个还是比较好理解

看看第三个start方法

参数可以看到很多啊 但是只调用了ZygoteProcess的start方法

![image-20240314092619964](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151029566.png)

ZygoteProcess类用于保持与Zygote进程的通信状态，进一步跟踪它的start方法

还没有到底 他又调用了startViaZygote方法

![image-20240314092734172](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151029567.png)

下面看看startViaZygote

![image-20240314093008532](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151029568.png)

可以看到它是新创建了一个字符串列表 然后将启动参数都传进去了

下面是书中截的关键部分

![image-20240314093054759](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151029569.png)

在注释1处创建了字符串列表argsForZygote，并将启动应用进程的启动参数保存在argsForZygote中，

方法的最后会调用zygoteSendArgsAndGetResult方法，需要注意的是，zygoteSendArgsAndGetResult方法的第一个参数中调用了openZygoteSocketIfNeeded方法①，而第二个参数是**保存应用进程的启动参数**的argsForZygote。

现在关键来到了zygoteSendArgsAndGetResult 也就是最后调用的那个函数

这个方法的主要作用是将传入应用的启动参数argsForZygote（刚刚的第二个参数）写入ZygoteState，这是ZygoteProcess的静态内部类，用于**表示与Zygote进程通信的状态。**

![image-20240314093234590](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151029570.png)

而刚开始进入的zygotestate其实是由openZygoteSocketIfNeeded返回的，来看看这个参数传入的方法具体做了什么

这里我初读代码看这个判断abi认为和zygote32or64相关，事实也确实如此。看看书中怎么说。

在2.2节讲到Zygote进程启动过程时我们得知，**在Zygote的main方法中会创建name为“zygote”的Server端Socket**。

在注释1处会调用ZygoteState的connect方法与名称为ZYGOTE_SOCKET的Socket建立连接，这里ZYGOTE_SOCKET的值为“zygote”，也就是说，**在注释1处与Zygote进程建立Socket连接，并返回ZygoteState类型的primaryZygoteState对象**（注意奥 **返回ZygoteState类型的primaryZygoteState对象**）

![image-20240314093635963](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151029571.png)

在注释2处如果primaryZygoteState与启动应用程序进程所需的ABI不匹配，则会在注释3处连接name为“zygote_secondary”的Socket。，而这里的abi又是哪来的呢，要往上看一些

![image-20240314094100941](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151029572.png)

现在来到注释三处的zygote_secondary，之前我们学到zygote为了适应不同的位数，所以采用了主副模式

name为“zygote”的为主模式，name 为“zygote_secondary”的为辅模式，那么注释2和注释3处的意思简单来说就是，如果连接Zygote主模式返回的ZygoteState与启动应用程序进程所需的ABI不匹配，则连接Zygote辅模式。如果在注释4处连接Zygote辅模式返回的ZygoteState与启动应用程序进程所需的ABI也不匹配，则抛出ZygoteStartFailedEx异常。

ok现在ams给zygote发送创建应用程序进程就结束了。

再来看一遍时序图

![image-20240314094409892](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151029573.png)

## Zygote接收请求并创建应用程序进程

发送完请求，要接收请求并创建了

时序图

![image-20240314094454365](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151029574.png)

Socket连接成功并匹配ABI后会返回ZygoteState类型对象，我们在分析zygoteSendArgsAndGetResult 方法中讲过，会将应用进程的启动参数argsForZygote写入ZygoteState中，这样Zygote进程就会收到一个创建新的应用程序进程的请求

这些参数以及在请求当中给出了，下面回到ZygoteInit的main方法

代码很长，用书中的

![image-20240314094537468](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151029575.png)

再讲一遍关于socket创建

在注释1处通过registerZygoteSocket方法创建了一个Server端的Socket，这个name 为**“zygote”的Socket用来等待AMS请求Zygote**，以创建新的应用程序进程

在注释2处预加载类和资源。在注释3处启动SystemServer进程，这样系统的服务也会由SystemServer进程启动起来。**在注释4处调用ZygoteServer的runSelectLoop方法来等待AMS请求创建新的应用程序进程**。

关键现在来到runSelectLoop

当有AMS的请求数据到来时，会调用注释2处的代码，结合注释1处的代码，我们得知注释2处的代码其实是调用ZygoteConnection的runOnce方法来处理请求数据

![image-20240314095036187](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151029576.png)

来到runonce

在注释1处调用readArgumentList方法来获取应用程序进程的启动参数，并在注释2处将readArgumentList方法返回的字符串数组args封装到Arguments类型的parsedArgs对象中。**在注释3处调用Zygote的forkAndSpecialize方法来创建应用程序进程，参数为parsedArgs中存储的应用进程启动参数，返回值为pid。**![image-20240314095131389](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151029577.png)

![image-20240314095136432](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151029578.png)

可以看到注释三的forkAndSpecialize很关键吧，forkAndSpecialize方法主要是通过fork当前进程来创建一个子进程的，如果pid等于0，则说明当前代码逻辑运行在新创建的子进程（应用程序进程）中，这时就会调用**handleChildProc**方法来处理应用程序进程，如下所示

![image-20240314095424300](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151029579.png)

handleChildProc方法中调用了ZygoteInit的zygoteInit方法，如下所示：

![image-20240314095501507](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151029580.png)

在注释2处调用了**RuntimeInit的applicationInit**方法：

![image-20240314095559622](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151029581.png)

在applicationInit方法中会在注释1处调用invokeStaticMain方法，需要注意的是，第一个参数args.startClass，它指的就是本章开头提到的参数android.app.ActivityThread。

来到这个invokeStaticMain

![image-20240314095649126](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151029582.png)

可以看到在注释1处通过反射获得了android.app.ActivityThread类，接下来在注释2处获得了ActivityThread的main方法，并将main方法传入注释3处的Zygote中的MethodAndArgsCaller类的构造方法中。

在注释3处抛出的MethodAndArgsCaller异常会**被Zygote的main方法**捕获

至于这里为何采用了抛出异常而不是直接调用ActivityThread的main方法，原理和本书2.3.1节Zygote处理SystemServer进程是一样的，这种抛出异常的处理会**清除所有的设置过程需要的堆栈帧**，并让**ActivityThread的main方法看起来像是应用程序进程的入口方法**。

看看这个zygote的main方法 可以看到调用了 **调用者的run方法**，也就是MethodAndArgsCaller的run方法

![image-20240315102505465](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151029583.png)

MethodAndArgsCaller这玩意是zygote.java的静态内部类

![image-20240315102619634](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151029584.png)

这里的mmethod指的是ActivityThread的main方法，这个main方法的invoke字后，ActivityThread的main方法就会被动态调用，然后就会进入activitythread的main方法。

于此 应用程序进程就创建好了并且**运行起来了主线程的管理类 ActivityThread**。

我们再回顾一下 这个时序图

![image-20240315102822520](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151029585.png)



















