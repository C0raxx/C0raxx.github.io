---
layout:     post
title:      【Android学习】《Android进阶解密》二之五
subtitle:   工具学习
date:       2024-03-13
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【Android进阶解密】
    - 【逆向】
    - 【Android】
---

# Android系统启动流程

结合之前的几个部分，可以总结出Android系统启动流程

## 1．启动电源以及系统启动

电源按下后，**引导芯片代码**从预定义的地方（固化在rom里面）开始执行，加载引导程序bootloader到ram开始执行

## 2.   引导程序BootLoader

是在Android操作系统开始运行前的一个小程序，它的**主要作用是把系统OS拉起来**并运行。

## 3．Linux内核启动

**当内核启动时，设置缓存、被保护存储器、计划列表、加载驱动。**当内核完成系统设置时，它首先在系统文件中寻找init.rc文件，并启动init进程。

## 4．init进程启动

上面找到了init.rc 开始启动init进程

## 5．Zygote进程启动

创建Java虚拟机并为Java虚拟机注册JNI方法，

创建服务器端Socket，

启动SystemServer进程。

## 6．SystemServer进程启动

启动Binder线程池

SystemServiceManager

并且启动各种系统服务。

## 7．Launcher启动

被SystemServer进程启动的**AMS**会启动Launcher，Launcher启动后会将已安装应用的快捷图标显示到界面上。

![image-20240314085316486](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151019602.png)



