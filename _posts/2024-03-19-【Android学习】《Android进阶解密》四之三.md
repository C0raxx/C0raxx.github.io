---
layout:     post
title:      【Android学习】《Android进阶解密》四之三
subtitle:   工具学习
date:       2024-03-19
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【Android进阶解密】
    - 【逆向】
    - 【Android】
---

# Service的绑定过程

之前是调用context的startservice启动那些service 现在通过Context的**bindService**来绑定Service，绑定Service的过程要比启动Service的过程复杂一些

## ContextImpl到AMS的调用过程

时序图

![image-20240321083814223](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403210914152.png)

首先是ContextWrapper调用到**ContextImpl的bindservice**，跟上节一样，这个mbase都是指向ContextImpl

![image-20240321084009058](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403210914155.png)

来到ContextImpl的bindservice

![image-20240321084116920](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403210914156.png)

需要关注的是这个返回出来的方法 bindservicecommon

如图

![image-20240321084144960](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403210914157.png)

里面有一些比较明显的获取apk信息的方法

在注释1处调用了LoadedApk类型的对象mPackageInfo的getServiceDispatcher方法，它的主要作用是**将ServiceConnection封装为IServiceConnection类型的对象sd**，从IServiceConnection的名字我们就能得知它实现了Binder机制，这样Service的绑定就**支持了跨进程。**

接着在注释2处我们又看见了熟悉的代码，最终会调用AMS的bindService方法。

## Service的绑定过程

开始前需要了解一些service相关的相关类型

- ServiceRecord：用于描述一个Service。
- ProcessRecord：一个进程的信息。
- ConnectionRecord：用于描述应用程序进程和Service建立的一次通信。
- AppBindRecord：应用程序进程通过Intent绑定Service时，会通过AppBindRecord来维护Service与应用程序进程之间的关联。其内部存储了谁绑定的Service （ProcessRecord）、被绑定的Service （AppBindRecord）、绑定Service的Intent （IntentBindRecord）和所有绑定通信记录的信息（ArraySet＜ConnectionRecord＞）。
- IntentBindRecord：用于描述绑定Service的Intent。

上面的进度来到了ams的bindservice当中

前半部分代码主要做一些异常判断，主要是主逻辑里面的返回函数

![image-20240321084408672](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403210914159.png)

接下来进入这个返回出来的bindServiceLocked

![image-20240321084734318](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403210914160.png)

1. 在注释1处调用了ServiceRecord的retrieveAppBindingLocked方法来获得AppBindRecord，retrieveAppBindingLocked 方法内部创建IntentBindRecord，并对IntentBindRecord的成员变量进行赋值，后面我们会详细介绍这个关键的方法。
2. 在注释2处调用bringUpServiceLocked方法，在bringUpServiceLocked方法中又调用realStartServiceLocked 方法，最终由ActivityThread 来调用Service的onCreate 方法启动Service，这也说明了bindService方法内部会启动Service，启动Service这一过程在4.2.2节中已经讲过，这里不再赘述。
3. 在注释3处s.app！=null 表示Service 已经运行，其中s 是ServiceRecord类型对象，app是ProcessRecord类型对象。b.intent.received表示当前应用程序进程已经接收到绑定Service 时返回的Binder，这样应用程序进程就可以通过Binder 来获取要绑定的Service的访问接口。
4. 在注释4处调用c.conn的connected方法，其中c.conn指的是IServiceConnection，它的具体实现为ServiceDispatcher.InnerConnection，其中ServiceDispatcher是LoadedApk的内部类，InnerConnection的connected方法内部会调用H的post方法向主线程发送消息，并且解决当前应用程序进程和Service跨进程通信的问题，在后面会详细介绍这一过程。
5. 在注释5处如果当前应用程序进程是第一个与Service进行绑定的，并且Service已经调用过onUnBind方法，
6. 则需要调用注释6处的代码。
7. 在注释7处如果应用程序进程的Client端没有发送过绑定Service的请求，则会调用注释8处的代码，注释8处和注释6处的代码区别就是最后一个参数rebind为false，表示不是重新绑定。

可以看到这个bindservicelocked大程度在高层次解决了service的绑定设定

下面需要深入进注释6的方法 requestServiceBindingLocked方法，代码如下

![image-20240321085320327](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403210914161.png)

上面注释一的i.requested表示是否发送过绑定Service的请求，从bindServiceLocked方法的**注释5处**得知是发送过的，因此，**！i.requested为false**。从bindServiceLocked方法的注释5处得知rebind值为true 那么（！i.requested||rebind）的值为true。

i.apps.size（）＞0表示什么呢？其中i是IntentBindRecord 类型的对象，AMS 会为每个绑定Service的Intent分配一个IntentBindRecord类型对象

我们来查看IntentBindRecord 类，不同的应用程序进程可能使用同一个Intent来绑定Service，因此在注释1处会用apps来存储所有用当前Intent绑定Service的应用程序进程。**i.apps.size（） ＞ 0表示所有用当前Intent绑定Service的应用程序进程个数大于0**

![image-20240321090118198](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403210914162.png)

下面来验证i.apps.size（）＞0是否为ture。我们回到bindServiceLocked方法的注释1处，ServiceRecord的retrieveAppBindingLocked方法如下所示

注意 是**上面的这个注释一**

![image-20240321090336196](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403210914163.png)

注释1处创建了IntentBindRecord，注释2处根据ProcessRecord获得IntentBindRecord中存储的AppBindRecord，如果AppBindRecord不为null就返回，如果为null就在注释3处**创建AppBindRecord，并将ProcessRecord作为key，AppBindRecord作为value保存在IntentBindRecord的apps（i.apps）**中。

回到requestServiceBindingLocked方法的注释1处，结合ServiceRecord的retrieveAppBindingLocked方法，**我们得知i.apps.size（）＞0为true**，这样就会调用注释2处的代码，r.app.thread的类型为IApplicationThread，它的实现我们已经很熟悉了，是ActivityThread的内部类ApplicationThread

scheduleBindService方法

首先将Service的信息封装成BindServiceData对象，BindServiceData的成员变量rebind的值为false，后面会用到它。接着将BindServiceData传入到sendMessage方法中。sendMessage向H发送消息

不得不说 这个H handle 在哪都有

![image-20240321090503215](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403210914165.png)

我们接着查看H的handleMessage方法：

![image-20240321090602397](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403210914166.png)

H 在接收到BIND_SERVICE类型消息时，会在handleMessage方法中会调用**handleBindService**方法

在注释1处**获取要绑定的Service**。

注释2处的BindServiceData的成员变量rebind的值为false，

这样会调用注释3处的代码来调用Service的onBind方法，到这里**Service处于绑定状态**了。如果rebind的值为true就会

调用注释5处的Service的onRebind方法，这一点结合前文的bindServiceLocked方法的注释5处，得出的结论就是：如果当前应用程序进程第一个与Service进行绑定，并且Service已经调用过onUnBind方法，则会调用Service的onRebind方法。handleBindService方法有两个分支，一个是绑定过Servive的情况，另一个是未绑定的情况，这里分析未绑定的情况，查看注释4处的代码，实际上是调用AMS的publishService方法。

这里基本上就是bindservice的主要部分

![image-20240321090640636](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403210914167.png)

这里给出 时序图

![image-20240321090844867](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403210914168.png)

还有一个publishservice方法没有吧

![image-20240321090914585](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403210914169.png)

这个publishservice调用了ActiveServices类型的mServices对象的**publishServiceLocked**方法

注释1处的代码在前面介绍过，**c.conn指的是IServiceConnection，它是ServiceConnection在本地的代理**，用于解决当前应用程序进程和Service跨进程通信的问题，具体实现为ServiceDispatcher.InnerConnection，其中ServiceDispatcher是LoadedApk的内部类

![image-20240321091002996](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403210914170.png)

，ServiceDispatcher.InnerConnection的connected方法的代码如下所示

![image-20240321091055532](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403210914171.png)



在注释1处调用了ServiceDispatcher类型的sd对象的connected方法，代码如下所示

![image-20240321091134883](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403210914173.png)

在注释1处调用Handler类型的对象mActivityThread的post方法，**mActivityThread实际上指向的是H**。因此，通过调用H的post方法将RunConnection对象的内容运行在主线程中。RunConnection是LoadedApk的内部类，定义如下所示：

![image-20240321091158309](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403210914174.png)

重点是里面的run方法

run方法中调用了doConnected方法

![image-20240321091237148](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403210914175.png)

在注释1处调用了ServiceConnection 类型的对象mConnection 的onServiceConnected方法，**这样在客户端实现了ServiceConnection接口类的onServiceConnected方法就会被执行。至此，Service 的绑定过程就分析完成。**

![image-20240321091320022](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403210914176.png)





























































