---
layout:     post
title:      【Android学习】《Android进阶解密》四之一
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

# 四大组件的工作过程

```
关联章节：第2章 Android系统启动；第3章 应用程序进程启动过程
```

书中的流程其实是很不错的，从android系统启动，到应用程序**进程**（还是要强调是进程），再到第四章的**启动应用程序**（这里才是应用程序）所需要的关于四大组件的知识，调理很清晰。

原话

```
在前面的两章中我们学习了系统的启动过程和应用进程的启动过程，应用进程启动后接着就该启动应用程序了，也就是启动根Activity。而Activity是四大组件之一，因此本章我们就来学习四大组件的工作过程。四大组件是应用开发最常接触的，包括Activity、Service、BroadcastReceiver和ContentProvider。
```

（这玩意和插件化也有联系）

为了不看晕，我把流程图贴一贴

Launcher请求AMS过程

![image-20240319104643187](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048980.png)

AMS 到ApplicationThread的调用过程

![image-20240319104659918](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048982.png)

ActivityThread启动Activity

![image-20240319104729255](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048983.png)



# 根Activity的启动过程

activity的启动方式有两种，一中是根activity的启动过程，一种是普通的activity启动过程，根activity一般指的是**应用程序进程启动后第一个加载的activity**

所以根activity的启动过程也可以理解为**应用程序的启动过程**

普通Activity指的是除应用程序启动的第一个Activity之外的其他Activity。



根activity的启动过程还是比较复杂的，因此分为三个部分来讲

**Launcher请求AMS过程**

**AMS到ApplicationThread的调用过程**

**ActivityThread启动Activity**

## Launcher请求AMS过程

之前已经了解过 在Android里面，launcher可以理解成桌面了，它把那些应用程序的快捷图标给显示到了桌面上面，那些应用程序的快捷图标就是 **根activity** 的入口

当我们点击某个应用程序的快捷图标时，会通过**launcher** 请求 **AMS** 来启动这个应用程序

不过这个过程也还是蛮坎坷的

![image-20240315114434947](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048984.png)

一步一步来吧，点击应用程序快捷图标的时候

调用launcher的startActivitySafely

![image-20240318103554861](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048986.png)

在注释一的地方将flag设置为Intent.FLAG_ACTIVITY_NEW_TASK，这样的话根activity会在新的任务栈当中启动

在注释2的地方 调用startactivity方法

在找定义的过程中 不知道为什么给我干这里来了，参数不对

![image-20240318104205574](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048987.png)

正常的应该是这里

里面根据options的值进行不同操作

![image-20240318104222410](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048988.png)

在startActivity 方法中会调用startActivityForResult 方法，它的第二个参数为-1，表示Launcher不需要知道Activity启动的结果

startActivityForResult 如下

这里注释一的mparent是activity类型的，表示当前activity的父类 如果目前根activity还没创建出来 所以进入第一个if 调用Instrumentation的execStartActivity方法，

Instrumentation 主要用来**监控应用程序和系统的交互**

![image-20240318104410427](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048989.png)

execStartActivity

![image-20240318104554705](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048990.png)

首先调用ActivityManager的getService方法来获取AMS的代理对象 ActivityManager的getService方法来获取AMS的代理对象 

```
这里与Android 8.0之前代码的逻辑有些不同，Android 8.0之前是通过ActivityManagerNative的getDefault来获取AMS的代理对象的，现在这个逻辑封装到了ActivityManager 中而不是ActivityManagerNative中
```

来查看ActivityManager的getService方法做了什么

![image-20240318104701154](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048992.png)

getService 方法调用了IActivityManagerSingleton的get方法，

我们接着往下看，IActivityManagerSingleton是一个Singleton类。

**在注释1处得到名为“activity”的Service引用，也就是IBinder类型的AMS的引用**。

接着在注释2处将它转换成IActivityManager类型的对象

```
这段代码采用的是AIDL，IActivityManager.java 类是由AIDL 工具在编译时自动生成的，IActivityManager.aidl 的文件路径为frameworks/base/core/java/android/app/IActivityManager.aidl。要实现进程间通信，服务器端也就是AMS只需要继承IActivityManager.Stub 类并实现相应的方法就可以了。注意Android 8.0之前并没有采用AIDL，而是采用了类似AIDL的形式，用AMS的代理对象ActivityManagerProxy来与AMS进行进程间通信，Android 8.0去除了ActivityManagerNative的内部类ActivityManagerProxy，代替它的是IActivityManager，它是AMS在本地的代理
```

只做科普吧，这里我的理解是这个launcher最后请求到ams 经过了层层的调用，最后通过本身调用到的iactivitymanager创建出自身对象的引用，并转换

同时在ams端实现对应的方法

其实我也不是很懂哈

看看时序图好了

![image-20240318105348772](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048993.png)

## AMS 到ApplicationThread的调用过程

来到part2

launcher请求完ams 代码逻辑焦点来到ams

接下来就是AMS到ApplicationThread的调用流程

时序图 看起来还是很恶心的

![image-20240318105502782](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048994.png)

在ams当中的startactivityasuser方法

可以看到里面返回调用了startActivityAsUser方法 

![image-20240318110456919](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048995.png)

比startActivity方法多了一个参数UserHandle.getCallingUserId（） startActivityAsUser方法会获得调用者的UserId

AMS根据这个UserId来确定调用者的权限

这里面就体现了如何判断权限

可以查看如下代码

![image-20240318111215563](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048996.png)

```
在注释1处判断调用者进程是否被隔离，如果被隔离则抛出SecurityException异常
在注释2处检查调用者是否有权限，如果没有权限也会抛出SecurityException异常。
```



startActivityLocked （明明是maywait。。）

最后调用了ActivityStarter的startActivityLocked 方法，startActivityLocked 方法的参数要比startActivityAsUser多几个，**需要注意的是倒数第二个参数类型为TaskRecord，代表启动的Activity所在的栈**。**最后一个参数"startActivityAsUser"代表启动的理由**。

![image-20240318111537467](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048997.png)

面这张图所在的类是7.0新加入的类 用来加载activity的控制类

会收集所有的逻辑来决定如何将Intent和Flags转换为Activity 并将Activity和Task以及Stack相关联

进一步进入这个startActivityLocked

首先是要判断启动理由是否为空

正常的话调用startactivity方法

![image-20240318111904302](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048998.png)

![image-20240318112005685](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048000.png)

上面的代码很长，逻辑多。

注释1处判断IApplicationThread类型的caller是否为null，**这个caller是方法调用一路传过来的，指向的是Launcher所在的应用程序进程的ApplicationThread对象**

在注释2处调用AMS的getRecordForAppLocked方法得到的是代表Launcher进程的callerApp对象，它是ProcessRecord类型的，ProcessRecord用于描述一个应用程序进程。同样地，ActivityRecord用于描述一个Activity，用来记录一个Activity 的所有信息。接下来创建ActivityRecord，用于描述将要启动的Activity，

并在注释3处将创建的ActivityRecord赋值给ActivityRecord[]类型的outActivity，这个outActivity会**作为注释4处的startActivity方法的参数**传递下去。

那下面来到这个startActivity

![image-20240318112152026](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048001.png)

紧接着调用了startActivityUnchecked

如下

下面这个方法主要处理与栈管理相关的

在标注①处我们得知，启动根Activity时会将Intent的Flag设置为FLAG_ACTIVITY_NEW_TASK，这样注释1处的条件判断就会满足，

接着执行注释2处的setTaskFromReuseOrCreateNewTask方法，其内部会创建一个新的TaskRecord，用来描述一个Activity任务栈，也就是说setTaskFromReuseOrCreateNewTask**方法内部会创建一个新的Activity任务栈**。Activity任务栈其实是一个假想的模型，并不真实存在，关于Activity 任务栈会在第6章进行介绍。

在注释3处会调用ActivityStackSupervisor的resumeFocusedStackTopActivityLocked方法

![image-20240318112227304](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048002.png)

resumeFocusedStackTopActivityLocked方法，如下所示：

![image-20240318113519936](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048003.png)

首先调用ActivityStack的topRunningActivityLocked 获取启动的Activity所在栈的栈顶的不是处于停止状态的ActivityRecord

感觉定语有点多  

在注释2处，如果ActivityRecord不为null，或者要启动的Activity的状态不是RESUMED状态，就会调用注释3处的ActivityStack的resumeTopActivityUncheckedLocked方法，**对于即将启动的Activity，注释2处的条件判断是肯定满足的**，我们来查看ActivityStack的resumeTopActivityUncheckedLocked方法

方法如下

![image-20240318114028626](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048004.png)

里面调用了注释1处ActivityStack的resumeTopActivityInnerLocked方法

resumeTopActivityInnerLocked代码很长 重点关注ActivityStackSupervisor的startSpecificActivityLocked方法即可

![image-20240318114058348](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048005.png)

startSpecificActivityLocked

![image-20240318114235820](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048006.png)

在注释1处获取即将启动的Activity所在的**应用程序进程**

在注释2处判断要启动的Activity所在的应用程序进程如果已经运行的话，就会调用注释3处的realStartActivityLocked方法，**这个方法的第二个参数是代表要启动的Activity所在的应用程序进程的ProcessRecord**。

看realStartActivityLocked

![image-20240318114322618](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048007.png)

这里的app.thread指的是**IApplicationThread**，它的实现是ActivityThread 的内部类ApplicationThread，其中ApplicationThread继承了**IApplicationThread.Stub**。

app指的是传入的要启动的Activity所在的应用程序进程，因此，这段代码指的**就是要在目标应用程序进程启动Activity**。当前代码逻辑运行在**AMS 所在的进程（SystemServer 进程）**中，通过ApplicationThread来与应用程序进程进行Binder通信，换句话说，ApplicationThread是**AMS所在进程（SystemServer进程）和应用程序进程的通信桥梁**

下面这个架构图能看得更加清晰一点

![image-20240318114511979](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048008.png)

再来看看时序图

![image-20240318114630368](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048009.png)

## ActivityThread启动Activity的过程

代码逻辑来到了**应用程序进程**当中了

时序图

![image-20240318114726716](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048010.png)

终于看到熟悉的oncreate了！

下面代码流程来到了ApplicationThread的scheduleLaunchActivity方法，这个ApplicationThread是activitythread的内部类 用来管理进程的主线程

下面来看这个ApplicationThread的scheduleLaunchActivity方法

sendMessage方法向H类发送类型为LAUNCH_ACTIVITY的消息，并将ActivityClientRecord传递过去

![image-20240319102922871](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048011.png)

代码流程来到activitythread的sendMessage

![image-20240319103021665](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048012.png)

这里mH指的是H，它是ActivityThread的内部类并继承自Handler，是应用程序进程中主线程的消息管理类。因为ApplicationThread是一个Binder，它的调用逻辑运行在Binder线程池中

来到binder线程池 当中这个函数

通过反射 调用的是handlemessage方法

这个方法当中对上面H.launcher_activity进行处理

```
在注释1处将传过来的msg的成员变量obj转换为ActivityClientRecord。
在注释2处通过getPackageInfoNoCheck方法获得LoadedApk类型的对象并赋值给ActivityClientRecord 的成员变量packageInfo。应用程序进程要启动Activity时需要将该Activity所属的APK加载进来，而LoadedApk就是用来描述已加载的APK文件的。
在注释3处调用handleLaunchActivity方法
```

![image-20240319103237350](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048013.png)

所以现在明白了 在handlemessage又调用会activitythread的handlelauncher方法了

```
注释1处的performLaunchActivity方法用来启动Activity，
注释2处的代码用来将Activity的状态设置为Resume。 
如果该Activity为null则会通知AMS停止启动Activity。 错误性检查
```

![image-20240319103538177](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048015.png)

于是 总算在这里 我们启动了activity

![image-20240319103704438](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048016.png)

看看这个关键的performlauncheractivity方法干嘛了

![image-20240319103730743](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048017.png)



![image-20240319103738862](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048018.png)

首先要说 如上是用来启动activity的 代码流程较长

1. 注释1处用来获取ActivityInfo，用于存储代码以及AndroidManifes设置的Activity和Receiver节点信息，比如Activity的theme和launchMode。
2. 在注释2处获取APK文件的描述类LoadedApk。
3. 在注释3处获取要启动的Activity的ComponentName类，在ComponentName 类中保存了该Activity的包名和类名。
4. 注释4处用来创建要启动Activity的**上下文环境**。
5. 注释5处根据ComponentName中存储的Activity类名，用类加载器来**创建该Activity的实例**。
6. 注释6处用来创建Application，makeApplication 方法内部会调用Application的onCreate方法。
7. 注释7处调用Activity的attach方法初始化Activity，**在attach方法中会创建Window对象（PhoneWindow）并与Activity自身进行关联**。
8. 在注释8处**调用Instrumentation的callActivityOnCreate方法来启动Activity**

callActivityOnCreate方法

![image-20240319104239684](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048019.png)

里面关键是注释一的performCreate方法

里面关键是oncreate方法

![image-20240319104303458](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048020.png)

在performCreate方法中会调用Activity的onCreate方法，讲到这里，根Activity就启动了，即应用程序就启动了

## 根activity启动涉及的进程

涉及4个进程，分别是Zygote进程、Launcher进程、AMS所在进程（SystemServer进程）、应用程序进程

![image-20240319104359700](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048021.png)

首先Launcher进程向AMS请求创建根Activity，AMS会判断根Activity所需的应用程序进程是否存在并启动，如果不存在就会请求Zygote进程创建应用程序进程。

应用程序进程启动后，AMS 会请求创建应用程序进程并启动根Activity。

**图4-5中步骤2采用的是Socket通信，步骤1和步骤4采用的是Binder通信。**

![image-20240319104529296](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403191048022.png)

如果是普通Activity启动过程会涉及几个进程呢？答案是**两个**，AMS所在进程和应用程序进程。实际上理解了根Activity的启动过程（根Activity的onCreate过程），根Activity和普通Activity其他生命周期状态（比如onStart、onResume等）过程也会很轻松掌握，这些知识点都是触类旁通的







