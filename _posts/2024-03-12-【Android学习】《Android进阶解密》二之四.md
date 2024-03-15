---
layout:     post
title:      【Android学习】《Android进阶解密》二之四
subtitle:   工具学习
date:       2024-03-12
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【Android进阶解密】
    - 【逆向】
    - 【Android】
---

# Launcher启动过程

上面的启动流程可以说是环环相扣的，

![image-20240313105249810](D:\Typora 1.6.7\Picture\image-20240313105249810.png)

![image-20240313105301800](D:\Typora 1.6.7\Picture\image-20240313105301800.png)

![image-20240313105307920](D:\Typora 1.6.7\Picture\image-20240313105307920.png)

那么这个launcher是哪里来的呢？

## 概述

系统启动的**最后一步**是启动一个应用程序来显示系统当中已经安装好的app，这玩意就是launcher，

Launcher在启动过程中会请求PackageManagerService返回系统中已经安装的应用程序的信息，并将**这些信息封装成一个快捷图标列表显示在系统屏幕**上，这样用户可以通过点击这些快捷图标来启动相应的应用程序。

## Launcher启动过程介绍

那么这个launcher是哪里来的呢？ 这是上面留的一个问题



时序图

![image-20240313111516578](D:\Typora 1.6.7\Picture\image-20240313111516578.png)

SystemServer进程在启动的过程中会启动PackageManagerService，PackageManagerService启动后会将系统中的应用程序安装完成。**在此前已经启动的AMS会将Launcher启动起来。**

之前对AMS还没什么概念，如下

```
AMS(ActivityManagerService)是Android中最核心的服务，主要负责系统中四大组件的启动、切换、调度及应用进程的管理和调度等工作，其职责与操作系统中的进程管理和调度模块相类似。
 AMS服务运行在system_server进程中，AMS由SystemServer的ServerThread线程创建。
```

启动launcher的入口为ams的systemready方法，在SystemServer的startOtherServices方法中被调用（这里的startOtherServices就是之前server启动的时候启动的其他服务）

看看这个startOtherServices里面的调用，关键如下

![image-20240313111759207](D:\Typora 1.6.7\Picture\image-20240313111759207.png)

进去看看这个方法做了什么 关键如下

![image-20240313111831424](D:\Typora 1.6.7\Picture\image-20240313111831424.png)

进一步跟踪 关键如下 调用ActivityStack的resumeTopActivityUncheckedLocked方法

ActivityStack对象是用来描述Activity堆栈的

![image-20240313111915741](D:\Typora 1.6.7\Picture\image-20240313111915741.png)

ActivityStack对象是用来描述Activity堆栈的 该方法的作用是**恢复位于活动堆栈顶部的活动，以便将其显示在屏幕上**。

![image-20240313111956825](D:\Typora 1.6.7\Picture\image-20240313111956825.png)

箭头处调用了resumeTopActivityInnerLocked，resumeTopActivityInnerLocked代码很长，仅截取关键部分 调用ActivityStackSupervisor的resumeHomeStackTask方法

```
当用户从后台切换回前台或者某个活动需要重新显示时，resumeTopActivityInnerLocked() 方法会被调用。它的主要功能是根据活动的状态和优先级，决定是否将活动显示在屏幕上，并调用相关的生命周期方法进行活动的恢复。此方法通常会在应用程序的任务栈中找到位于栈顶的活动，然后调用其 onResume() 方法将其展示在前台。
```

![image-20240313112157587](D:\Typora 1.6.7\Picture\image-20240313112157587.png)

其中调用了AMS的startHomeActivityLocked方法 其中又调用了startHomeActivityLocked

![image-20240313112406021](D:\Typora 1.6.7\Picture\image-20240313112406021.png)

startHomeActivityLocked如下，可以看到回到了ams.java

![image-20240313112517828](D:\Typora 1.6.7\Picture\image-20240313112517828.png)

注释1处的mFactoryTest代表系统的运行模式，系统的运行模式分为三种，分别是非工厂模式、低级工厂模式和高级工厂模式，mTopAction 则用来描述第一个被启动Activity组件的Action，它的默认值为Intent.ACTION_MAIN。因此注释1处的代码的意思就是mFactoryTest 为FactoryTest.FACTORY_TEST_LOW_LEVEL （低级工厂模式）并且mTopAction等于null时，直接返回false。



注释2处的getHomeIntent方法如下所示：

![image-20240313113054241](D:\Typora 1.6.7\Picture\image-20240313113054241.png)

在getHomeIntent方法中创建了Intent，并将mTopAction和mTopData传入。mTopAction的值为Intent.ACTION_MAIN，并且如果系统运行模式不是低级工厂模式，则将intent的Category 设置为Intent.CATEGORY_HOME，最后返回该Intent，再回到AMS的startHomeActivityLocked 方法，假设系统的运行模式不是低级工厂模式，在注释3处判断符合Action为Intent.ACTION_MAIN、Category为Intent.CATEGORY_HOME的应用程序是否已经启动，如果没启动则调用注释4处的方法启动该应用程序。**这个被启动的应用程序就是Launcher，因为Launcher的AndroidManifest文件中的intent-filter标签匹配了Action为Intent.ACTION_MAIN，Category为Intent.CATEGORY_HOME。**

![image-20240313114351401](D:\Typora 1.6.7\Picture\image-20240313114351401.png)

我们回到之前这里

我们得知如果Launcher没有启动就会调用ActivityStarter的startHomeActivityLocked方法来启动Launcher

![image-20240313114525124](D:\Typora 1.6.7\Picture\image-20240313114525124.png)

进入的是如下方法

![image-20240313114542782](D:\Typora 1.6.7\Picture\image-20240313114542782.png)

在注释1处将Launcher放入HomeStack中，HomeStack是在ActivityStackSupervisor中定义的用于存储Launcher的变量。接着调用startActivityLocked方法来启动Launcher，剩余的过程会和Activity的启动过程类似

最终进入Launcher的onCreate方法中，到这里Launcher就完成了启动。后面和activity启动一起学好了

回到时序图

![image-20240313114907260](D:\Typora 1.6.7\Picture\image-20240313114907260.png)

## Launcher中应用图标显示过程

现在启动完成

作为桌面它会显示应用程序图标，这与应用程序开发有所关联，应用程序图标是用户进入应用程序的入口，因此我们有必要了解Launcher是如何显示应用程序图标的。

看看launcher的oncreate

![image-20240313114945824](D:\Typora 1.6.7\Picture\image-20240313114945824.png)

在注释1处获取LauncherAppState的实例

在注释2处调用它的**setLauncher**方法并将Launcher对象传入

setLauncher如下 关键是这个初始化函数

![image-20240314083918624](../Typora 1.6.7/Picture/image-20240314083918624.png)

initialize

![image-20240314083934607](../Typora 1.6.7/Picture/image-20240314083934607.png)

这里传入的launcher被封装成弱引用对象mcallbacks



现在从之前注释2跳出来 回到注释三的地方

在注释3处调用了LauncherModel的startLoader方法

![image-20240314084133930](../Typora 1.6.7/Picture/image-20240314084133930.png)

在注释1处创建了具有消息循环的线程HandlerThread对象。

在注释2处创建了Handler，并且传入HandlerThread的Looper，这里Hander的作用就是向HandlerThread发送消息。

在注释3处创建**LoaderTask**，

在注释4处将**LoaderTask**作为消息发送给HandlerThread。

loaderTask类实现了Runnable接口，当LoaderTask所描述的消息被处理时，则会调用它的run方法，LoaderTask是LauncherModel的内部类

![image-20240314084249264](../Typora 1.6.7/Picture/image-20240314084249264.png)



这里就设计显示应用的原理了

```
Launcher 是用工作区的形式来显示系统安装的应用程序的快捷图标的，每一个工作区都是用来描述一个抽象桌面的，它由n个屏幕组成，每个屏幕又分为n个单元格，每个单元格用来显示一个应用程序的快捷图标。
```

这注释一的地方用到了之前所涉及到的mcallback，具体是调用callbacks的bindAllApplications，从之前我们得知这个callbacks实际是指向Launcher的

关于bindAllApplications

![image-20240314084448788](../Typora 1.6.7/Picture/image-20240314084448788.png)

可以看到注释三中调用run方法调用到bindAllApplications，其中注释1处会调用AllAppsContainerView类型的mAppsView对象的setApps方法，**并将包含应用信息的列表apps传进去**

看看setapps

![image-20240314084552784](../Typora 1.6.7/Picture/image-20240314084552784.png)



里面把包含应用信息的列表apps设置给mapps 这个mApps是AlphabeticalAppsList类型的对象

接着查看AllAppsContainerView的onFinishInflate方法如下所示

![image-20240314084733280](../Typora 1.6.7/Picture/image-20240314084733280.png)

onFinishInflate方法会在AllAppsContainerView加载完XML布局时调用，在注释1处得到AllAppsRecyclerView用来显示App列表，并在注释2处将此前的mApps设置进去，在注释3处为AllAppsRecyclerView设置Adapter。这样显示应用程序快捷图标的列表就会显示在屏幕上。

ok结束了























