---
layout:     post
title:      【Android学习】《Android进阶解密》二之二
subtitle:   工具学习
date:       2024-03-11
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【Android进阶解密】
    - 【逆向】
    - 【Android】
---

## Zygote进程启动过程

init进程启动过程，在启动过程中主要做了三件事，其中一件就是创建了Zygote 进程，本节接着学习Zygote 进程启动过程，首先我们要了解Zygote是什么。

### 概述

DVM,ART,应用程序进程以及运行系统的关键服务的systemserver进程都是zygote进程来创建出来的，所以也叫他孵化器

它通过fork形式创建应用程序进程和systemserver进程

*由于Zygote进程在启动时会创建DVM或者ART，因此通过fock而创建的应用程序进程和SystemServer进程可以在内部获取一个DVM或者ART的实例副本。*

起初Zygote进程的名称并不是叫“zygote”，而是叫“app_process”

### Zygote启动脚本

书接上回init rc用import语句引入了zygote启动脚本

也表示不会直接引入固定的文件 而是根据属性ro.zygote引入不同的文件

```
从Android 5.0开始，Android开始支持64位程序，Zygote也就有了32位和64位的区别，所以在这里用ro.zygote属性来控制使用不同的Zygote启动脚本，从而也就启动了不同版本的Zygote进程
```

![image-20240311092503747](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403120944449.png)

分为四种

![image-20240311092604156](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403120944450.png)

init.zygote32.rc
执行程序为app_process，classname为main，如果audioserver、cameraserver、media等进程终止了

```
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    class main
    priority -20
    user root
    group root readproc
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart netd
    onrestart restart wificond
    writepid /dev/cpuset/foreground/tasks
```

init.zygote32_64.rc

```
service zygote /system/bin/app_process32 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
    class main
    priority -20
    user root
    group root readproc
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart netd
    onrestart restart wificond
    writepid /dev/cpuset/foreground/tasks

service zygote_secondary /system/bin/app_process64 -Xzygote /system/bin --zygote --socket-name=zygote_secondary
    class main
    priority -20
    user root
    group root readproc
    socket zygote_secondary stream 660 root system
    onrestart restart zygote
    writepid /dev/cpuset/foreground/tasks
```

有两个Service类型语句，说明会启动两个Zygote进程，一个名称为zygote，执行程序为app_process32，作为主模式；另一个名称为zygote_secondary，执行程序为app_process64，作为辅模式。

还有俩类似 不写了

### Zygote进程启动过程介绍

承接二之一的内容 init启动zygote（那条action语句)调用app_main.cpp的main函数中的AppRuntime的start方法来启动Zygote进程的

![image-20240311093145218](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403120944451.png)

从app main的main开始

![image-20240311094644522](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403120944452.png)

在注释5处，如果zygote为true，就说明当前运行在Zygote进程中，就会调用注释6处的**AppRuntime**的start函数

代码很长 找关键的 说实话不太明白最后是怎么找到zygoteinit的

![image-20240311095057901](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403120944453.png)

![image-20240311095105601](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403120944454.png)

```
在注释1处调用startVm函数来创建Java虚拟机，在注释2处调用startReg函数为Java虚拟机注册JNI方法。注释3处的className的值是传进来的参数，它的值为com.android.internal.os.ZygoteInit。在注释4处通过toSlashClassName函数，将className的“.”替换为“/”，替换后的值为com/android/internal/os/ZygoteInit，并赋值给slashClassName，接着在注释5处根据slashClassName找到ZygoteInit，找到了ZygoteInit后顺理成章地在注释6处找到ZygoteInit的main方法。最终会在注释7处通过JNI调用ZygoteInit的main方法。这里为何要使用JNI呢？因为ZygoteInit的main方法是由Java语言编写的，当前的运行逻辑在Native中，这就需要通过JNI来调用Java。这样Zygote就从Native层进入了Java框架层。
```

好像是如下的地方传进去的

![image-20240311095223903](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403120944455.png)

在我们通过JNI调用ZygoteInit的main方法后，Zygote便进入了Java框架层，此前是没有任何代码进入Java框架层的，换句话说是Zygote开创了Java框架层。

看看这个zygote的main方法

![image-20240311095511748](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403120944456.png)

在注释1处通过registerServerSocket 方法来创建一个Server端的Socket，这个name为“zygote”的Socket用于等待ActivityManagerService请求Zygote来创建新的应用程序进程

在注释2处预加载类和资源。在注释3处启动SystemServer进程，这样系统的服务也会由SystemServer进程启动起来。在注释4处调用ZygoteServer的runSelectLoop方法来等待AMS请求创建新的应用程序进程。

所以zygoteinit的main做了这些事

- （1）创建一个Server端的Socket。
- （2）预加载类和资源。
- （3）启动SystemServer进程。
- （4）等待AMS请求创建新的应用程序进程

#### 创建socketregisterZygoteSocket

![image-20240311095740792](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403120944457.png)

在注释6处创建LocalServerSocket，也就是服务器端的Socket，并将文件操作符作为参数传进去。在Zygote进程将SystemServer进程启动后，就会在这个服务器端的Socket上等待AMS请求Zygote进程来创建新的应用程序进程。

#### 启动SystemServer进程

```
private static boolean startSystemServer(String abiList, String socketName, ZygoteServer zygoteServer)
            throws Zygote.MethodAndArgsCaller, RuntimeException { 这玩意创建启动systemserver的启动参数
        long capabilities = posixCapabilitiesAsBits(
            OsConstants.CAP_IPC_LOCK,
            OsConstants.CAP_KILL,
            OsConstants.CAP_NET_ADMIN,
            OsConstants.CAP_NET_BIND_SERVICE,
            OsConstants.CAP_NET_BROADCAST,
            OsConstants.CAP_NET_RAW,
            OsConstants.CAP_SYS_MODULE,
            OsConstants.CAP_SYS_NICE,
            OsConstants.CAP_SYS_PTRACE,
            OsConstants.CAP_SYS_TIME,
            OsConstants.CAP_SYS_TTY_CONFIG,
            OsConstants.CAP_WAKE_ALARM
        );
        /* Containers run without this capability, so avoid setting it in that case */
        if (!SystemProperties.getBoolean(PROPERTY_RUNNING_IN_CONTAINER, false)) {
            capabilities |= posixCapabilitiesAsBits(OsConstants.CAP_BLOCK_SUSPEND);
        }
        /* Hardcoded command line to start the system server */
        String args[] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1023,1032,3001,3002,3003,3006,3007,3009,3010",
            "--capabilities=" + capabilities + "," + capabilities,
            "--nice-name=system_server",
            "--runtime-args",
            "com.android.server.SystemServer",
        };
        ZygoteConnection.Arguments parsedArgs = null;

        int pid;

        try {
            parsedArgs = new ZygoteConnection.Arguments(args);
            ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
            ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);

            /* Request to fork the system server process */
            pid = Zygote.forkSystemServer( 创建一个子进程 就是systemserver的进程
                    parsedArgs.uid, parsedArgs.gid,
                    parsedArgs.gids,
                    parsedArgs.debugFlags,
                    null,
                    parsedArgs.permittedCapabilities,
                    parsedArgs.effectiveCapabilities);
        } catch (IllegalArgumentException ex) {
            throw new RuntimeException(ex);
        }

        /* For child process */
        if (pid == 0) {
            if (hasSecondZygote(abiList)) {
                waitForSecondaryZygote(socketName);
            }

            zygoteServer.closeServerSocket();
            handleSystemServerProcess(parsedArgs);处理systemserver进程
            
        }

        return true;
    }
```

注释1处的代码用来创建args数组，这个数组用来保存启动SystemServer的启动参数，其中可以看出SystemServer进程的用户id 和用户组id被设置为1000，并且拥有用户组1001～1010、1018、1021、1032、3001～3010的权限；进程名为system_server；启动的类名为com.android.server.SystemServer。在注释2处将args数组封装成Arguments对象并供注释3处的forkSystemServer函数调用。在注释3处调用Zygote的forkSystemServer方法，其内部会调用nativeForkSystemServer 这个Native 方法，nativeForkSystemServer方法最终会通过fork函数在当前进程创建一个子进程，也就是SystemServer 进程，如果forkSystemServer方法返回的pid的值为0，就表示当前的代码运行在新创建的子进程中，则执行注释4处的handleSystemServerProcess来处理SystemServer进程

#### runSelectLoop

其实也就是这个

![image-20240312075323359](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403120944458.png)

启动SystemServer进程后，会执行ZygoteServer的runSelectLoop方法，如下所示：

loop好像是个什么环来着

![image-20240312075506235](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403120944459.png)

注释1处的mServerSocket就是我们在registerZygoteSocket函数中创建的服务器端Socket，调用mServerSocket.getFileDescriptor（）函数用来获得该Socket的fd字段的值并添加到fd列表fds中。

在注释2处通过遍历将fds存储的信息转移到pollFds数组中。

在注释3处对pollFds进行遍历，如果i==0，说明服务器端Socket 与客户端连接上了，换句话说就是，当前Zygote进程与AMS建立了连接。在注释4处通过acceptCommandPeer方法得到ZygoteConnection类并添加到Socket连接列表peers中，接着将该ZygoteConnection的fd添加到fd列表fds中，以便可以接收到AMS发送过来的请求。

如果i的值不等于0，则说明AMS向Zygote进程发送了一个创建应用进程的请求，则在注释5处调用ZygoteConnection的runOnce函数来创建一个新的应用程序进程，并在成功创建后将这个连接从Socket连接列表peers和fd列表fds中清除。

### Zygote进程启动总结

- （1）创建AppRuntime并调用其start方法，启动Zygote进程。
- （2）创建Java虚拟机并为Java虚拟机注册JNI方法。
- （3）通过JNI调用ZygoteInit的main函数进入Zygote的Java框架层。
- （4）通过registerZygoteSocket方法创建服务器端Socket，并通过runSelectLoop方法等待AMS的请求来创建新的应用程序进程。
- （5）启动SystemServer进程。

















































































































































