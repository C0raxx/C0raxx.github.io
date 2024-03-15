---
layout:     post
title:      【Android学习】《Android进阶解密》三之三
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

# Binder 线程池启动过程

在之前应用程序进程创建的过程中 有涉及到关于binder线程池创建的问题。

关于调用时机在这

![image-20240315103547192](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151054909.png)

关于详细调用代码是在这里面创建binder线程池的

![image-20240315103616594](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151054911.png)

我们进入nativezygoteinit看看

可以看到是一个native层的jni方法

![image-20240315103721182](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151054912.png)

得到java层的函数名还不够 还需要得到他在jni中的函数

可以在AndroidRuntime.cpp的JNINativeMethod数组中我们得知它对应的函数是com_android_internal_os_ZygoteInit_nativeZygoteInit

![image-20240315103842049](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151054913.png)

那接下来就是要找com_android_internal_os_ZygoteInit_nativeZygoteInit

全局搜 找到

![image-20240315103938268](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151054914.png)

这个gcurruntime是在android初始化的时候就创建完的

AppRuntime继承自AndroidRuntime，AppRuntime创建时就会**调用AndroidRuntime的构造函数，gCurRuntime就会被初始化**，**它指向的是AppRuntime**

![image-20240315104201821](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151054915.png)

书上说 下面就会调用到appruntime的onzygoteinit函数，appruntime在appmain里面实现

![image-20240315104644844](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403151054916.png)

最后一行调用**ProcessState的startThreadPool函数**来**启动Binder线程池**

代码不长 逻辑就是先检查mThreadPoolStarted是不是false（这玩意初始值为false），如果不是，就进入if里面的语句 关键是执行spawnPooledThread(true);

```
void ProcessState::startThreadPool()
{
    AutoMutex _l(mLock);
    if (!mThreadPoolStarted) {   1
        mThreadPoolStarted = true;   2
        spawnPooledThread(true);
    }
}
```

跟踪spawnPooledThread 发现代码也不长

简单分析下 先获取名称 然后新建对象t，调用这个t的run方法启动一个新的线程，注意这个是PoolThread类

```
void ProcessState::spawnPooledThread(bool isMain)
{
    if (mThreadPoolStarted) {
        String8 name = makeBinderThreadName();
        ALOGV("Spawning new pooled thread, name=%s\n", name.string());
        sp<Thread> t = new PoolThread(isMain);
        t->run(name.string());
    }
}

```

着手点就在于这个PoolThread做了什么

可以找到的是这个poolthread继承自thread类，然后注释一处调用IPCThreadState的joinThreadPool函数，**将当前线程注册到Binder驱动程序中**

这样新创建的应用程序进程可以支持binder进程间通讯了

```
class PoolThread : public Thread
{
public:
    explicit PoolThread(bool isMain)
        : mIsMain(isMain)
    {
    }
    
protected:
    virtual bool threadLoop()
    {
        IPCThreadState::self()->joinThreadPool(mIsMain);  1
        return false;
    }
    
    const bool mIsMain;
};
```





**我们只需要创建当前进程的Binder对象，并将它注册到ServiceManager中就可以实现Binder进程间通信**



。。没有时序图

























































































