---
layout:     post
title:      【Android学习】《Android进阶解密》四之二
subtitle:   工具学习
date:       2024-03-18
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【Android进阶解密】
    - 【逆向】
    - 【Android】
---

# Service的启动过程

这部分与上一部分 根activity启动有部分相似

另外Service启动过程涉及上下文**Context**的知识点，这里只关注流程而不会详细介绍Contex

Service的启动过程将分为两个部分来进行讲解，分别是

**ContextImpl到ActivityManageService**的调用过程

和

**ActivityThread启动Service**

## ContextImpl到AMS的调用过程

过程不是很长

流程图

![image-20240319110305399](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191130298.png)

如果要启动service 需要调用ContextWrapper中的startService

可以看到好像是继承自context的

调用mBase的startService方法，**Context类型的mBase对象具体指的是什么呢**

![image-20240319110418221](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191130299.png)

在**四之一**学过，ActivityThread启动Activity时会调用如下代码创建Activity的上下文环境

这个注释一处创建context 然后绑定activity和context

![image-20240319110728147](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191130300.png)

这个上下文对象**appContext的具体类型是什么**？我们接着查看createBaseContextForActivity方法

![image-20240319110833061](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191130301.png)

其中的**appcontext的具体类型ContextImpl**，在Activity的attach方法中将ContextImpl赋值给ContextWrapper的成员变量mBase，、

**因此，上面提出的问题就得到了解答，mBase具体指向的就是ContextImpl。**

所以上面的就变成了ContextImpl.startservice了

紧接着来查看ContextImpl的startService方法

![image-20240319111000902](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191130302.png)

在startService方法中会返回startServiceCommon方法，在startServiceCommon方法中会在注释1处调用AMS的代理IActivityManager的startService方法，**最终调用的是AMS的startService方法**

只要晓得最后调用到AMS的startservice就可以了

## ActivityThread启动Service

这玩意相对复杂一点

时序图

其实说实话 感觉和activity的创建还蛮像的

![image-20240319111224781](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191130303.png)

查看AMS的startService方法

调用mServices的startServiceLocked方法，mServices的类型是ActiveServices，

![image-20240319111317217](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191130304.png)

ActiveServices的startServiceLocked方法代码如下所示：

![image-20240319112007305](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191130305.png)

1. 注释1处的retrieveServiceLocked方法会查找是否有与参数service对应的ServiceRecord，如果没有找到，就会**调用PackageManagerService去获取参数service对应的Service信息，并封装到ServiceRecord中**，最后将ServiceRecord封装为ServiceLookupResult返回。其中ServiceRecord用于描述一个Service，和此前讲过的ActivityRecord类似。
2. 在注释2处通过注释1处返回的ServiceLookupResul**t得到参数service对应的ServiceRecord**，并传入到注释3处的startServiceInnerLocked方法中。

下面来到startServiceInnerLocked

![image-20240319112105821](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191130306.png)

又调用了bringUpServiceLocked方法
这玩意更长一点

![image-20240319112123589](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191130307.png)

![image-20240319112129922](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191130308.png)

1. 在注释1处得到ServiceRecord 的processName值并赋给procName，其中processName用来描述Service想要在哪个进程中运行，默认是当前进程，我们也可以在AndroidManifest文件中设置android：process 属性来新开启一个进程运行Service。
2. 在注释2处将procName和Service的uid传入到AMS的getProcessRecordLocked 方法中，查询是否存在一个与Service对应的ProcessRecord类型的对象app，ProcessRecord主要用来描述运行的应用程序进程的信息。
3. 在注释5处判断Service对应的app为null则说明用来运行Service的应用程序进程不存在，
4. 则调用注释6处的AMS的startProcessLocked方法来创建对应的应用程序进程，关于创建应用程序进程请查看第3章的内容，**这里只讨论没有设置android：process属性**，即应用程序进程存在的情况。
5. 在注释3处判断如果用来运行Service的应用程序进程存在，
6. 则调用注释4处的**realStartServiceLocked**方法启动service



关键在于realStartServiceLocked

![image-20240319112500451](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191130309.png)

其中调用了app.thread的scheduleCreateService方法。

类似activity

其中app.thread是IApplicationThread 类型的，它的实现是ActivityThread的内部类ApplicationThread。ApplicationThread的scheduleCreateService方法如下所示

![image-20240319112557216](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191130310.png)

scheduleLaunchActivity方法将启动Service的参数封装成ActivityClientRecord，sendMessage方法向**H类发送类型为CREATE_SERVICE的消息，并将ActivityClientRecord**传递过去，这个过程和4.1.3节ActivityThread启动Activity的过程是**类似**的。sendMessage方法有多个重载方法，

最终调用的sendMessage方法如下所示

![image-20240319112814392](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191130311.png)

这里的mh指的是H，它是ActivityThread的内部类并继承自**Handler **是应用程序进程中主线程的消息管理类

查看H的handleMessage方法：

![image-20240319112850818](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191130312.png)

handleMessage 方法根据消息类型为CREATE_SERVICE，会调用handleCreateService方法：

![image-20240319112942314](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191130313.png)

可以看到是一些具体的创建service流程了

在注释1处获取要启动Service的应用程序的LoadedApk，LoadedApk是一个APK文件的描述类。

在注释2处通过调用LoadedApk的getClassLoader方法来获取类加载器。

接着在注释3处根据CreateServiceData对象中存储的Service信息，创建Service实例。

在注释4处创建Service的上下文环境ContextImpl对象。

在注释5处通过Service的attach方法来初始化Service。

在注释6处调用Service的onCreate方法，这样Service就启动了。

在注释7处将启动的Service加入到ActivityThread的成员变量mServices中，其中mServices是ArrayMap类型。

结束 再放一遍这个时序图

![image-20240319113145428](D:\Typora 1.6.7\Picture\image-20240319113145428.png)

![image-20240319113155130](D:\Typora 1.6.7\Picture\image-20240319113155130.png)









































