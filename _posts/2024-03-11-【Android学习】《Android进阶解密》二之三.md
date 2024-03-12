---
layout:     post
title:      【Android学习】《Android进阶解密》二之三
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

# SystemServer处理过程

SystemServer进程主要用于创建系统服务，我们熟知的AMS、WMS和PMS都是由它来创建的

## Zygote处理SystemServer进程

![image-20240312083757098](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403120943698.png)

startSystemServer方法中启动了SystemServer进程

部分代码

![image-20240312083854054](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403120943700.png)

SystemServer 进程复制了Zygote进程的地址空间，因此也会得到Zygote进程创建的Socket，这个Socket对于SystemServer进程**没有用处**，因此，需要注释1处的代码来关闭该Socket

而注释2处的方法是用来启动这个systemserver进程的

![image-20240312084022218](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403120943701.png)

在注释1处创建了PathClassLoader 这个pathclassloader和后面的dexclassloader有点像，都用于加载dex，但还有些区别。

![image-20240312084426632](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403120943702.png)

然后调用zygoteinit方法 代码如下

![image-20240312084502272](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403120943704.png)

在注释1处调用nativeZygoteInit方法，一看方法的名称就知道调用的是Native层的代码，**用来启动Binder线程池**，这样SystemServer进程就可以使用Binder与其他进程进行通信了

注释2处是用于进入SystemServer的main方法，现在分别对注释1和注释2的内容进行介绍。

### 启动binder线程池

这玩意是一个native函数，如果要找到它对应的jni文件，需要通过jni的gmethods数组 找到这玩意对应的是AndroidRuntime.cpp的com_android_internal_os_ZygoteInit_nativeZygoteInit函数：

```
int register_com_android_internal_os_ZygoteInit(JNIEnv* env)
{
    const JNINativeMethod methods[] = {
        { "nativeZygoteInit", "()V",
            (void*) com_android_internal_os_ZygoteInit_nativeZygoteInit },
    };
    return jniRegisterNativeMethods(env, "com/android/internal/os/ZygoteInit",
        methods, NELEM(methods));
}
```

可以找到这个native对应的是com_android_internal_os_ZygoteInit_nativeZygoteInit

```
static void com_android_internal_os_ZygoteInit_nativeZygoteInit(JNIEnv* env, jobject clazz)
{
    gCurRuntime->onZygoteInit();
}
```

gCurRuntime是Androidruntime类型的指针，指向其子类appruntime，这里指向的是onZygoteInit

指向这里

```
virtual void onZygoteInit()
    {
        sp<ProcessState> proc = ProcessState::self();
        ALOGV("App process: starting thread pool.\n");
        proc->startThreadPool();
    }
```

proc->startThreadPool(); 这玩意用来启动一个线程池，这样SystemServer进程就可以使用Binder与其他进程进行通信了

所以从开始的zygoteinit的那条nativezygoteinit用来调用RuntimeInit.java的nativeZygoteInit函数 再往下启动binder线程池



### 进入SystemServer的main方法

![image-20240312090248654](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403120943705.png)

代码很长 关键在这

![image-20240312090455480](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403120943706.png)

进来之后

![image-20240312090647486](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403120943707.png)

注释1处的className为com.android.server.SystemServer，通过反射返回的cl为SystemServer类。在注释2处找到SystemServer中的main方法。在注释3处将找到的main方法传入MethodAndArgsCaller异常中并抛出该异常，捕获MethodAndArgsCaller异常的代码在ZygoteInit.java的main方法中，这个main方法会调用SystemServer的main方法。那么为什么不直接在invokeStaticMain方法中调用SystemServer的main方法呢？原因是这种抛出异常的处理会清除所有的设置过程需要的堆栈帧，并让SystemServer的main方法看起来像是SystemServer进程的入口方法。在Zygote启动了SystemServer进程后，SystemServer进程已经做了很多的准备工作，而这些工作都是在SystemServer的main方法调用之前做的，这使得SystemServer的main方法看起来不像是SystemServer进程的入口方法，而这种抛出异常交由ZygoteInit.java的main方法来处理，会让SystemServer的main方法看起来像是SystemServer进程的入口方法。

总结一下 在进入这里之前就已经做了很多system的准备工作，抛出异常来调用main方法显得main像systemserver进程入口方法

在ZygoteInit.java的main方法中有关捕获MethodAndArgsCaller



到这里就会捕获

![image-20240312091144868](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403120943708.png)

捕获到之后会调用MethodAndArgsCaller的run方法，MethodAndArgsCaller是Zygote.java的静态内部类：

```
    public static class MethodAndArgsCaller extends Exception
            implements Runnable {
        /** method to call */
        private final Method mMethod;

        /** argument array */
        private final String[] mArgs;

        public MethodAndArgsCaller(Method method, String[] args) {
            mMethod = method;
            mArgs = args;
        }

        public void run() {
            try {
                mMethod.invoke(null, new Object[] { mArgs });
            } catch (IllegalAccessException ex) {
                throw new RuntimeException(ex);
            } catch (InvocationTargetException ex) {
                Throwable cause = ex.getCause();
                if (cause instanceof RuntimeException) {
                    throw (RuntimeException) cause;
                } else if (cause instanceof Error) {
                    throw (Error) cause;
                }
                throw new RuntimeException(ex);
            }
        }
    }
```

MethodAndArgsCaller是zygote.java的静态内部类 run方法当中比较关键的是

` mMethod.invoke(null, new Object[] { mArgs });`

调用完上面的语句之后，systemserver的main方法就会被动态调用，SystemServer 进程就进入了SystemServer的main方法中。

## 解析SystemServer进程

既然进入了SystemServer的main方法当中了

![image-20240312091948221](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403120943710.png)

到了这里

```
public static void main(String[] args) {
        new SystemServer().run();
    }
```

调用了他的run方法

run方法很长 截书

![image-20240312092147934](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403120943711.png)

在注释1处加载了动态库libandroid_servers.so。接下来在注释2处创建SystemServiceManager，它会对系统服务进行创建、启动和生命周期管理。

在注释3处的startBootstrapServices方法中用SystemServiceManager启动了ActivityManagerService、PowerManagerService、PackageManagerService等服务。在注释4处的startCoreServices方法中则启动了DropBoxManagerService、BatteryService、UsageStatsService和WebViewUpdateService。在注释5处的startOtherServices 方法中启动了CameraService、AlarmManagerService、VrManagerService等服务。这些服务的父类均为SystemService。

总而言之，之三个starxxxservice创建了各种服务，也可以官方把系统服务分为三类分别是引导服务、核心服务和其他服务 

部分

![image-20240312092648432](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403120943712.png)

具体函数例如下面这个

![image-20240312092954252](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403120943713.png)

调用了

```
private static void traceBeginAndSlog(String name) {
        Slog.i(TAG, name);
        BOOT_TIMINGS_TRACE_LOG.traceBegin(name);
    }

```

这里不关键 看另外一个

![image-20240312093228126](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403120943714.png)



来到这里

```
public void startService(@NonNull final SystemService service) {
        // Register it.
        mServices.add(service);
        // Start it.
        long time = System.currentTimeMillis();
        try {
            service.onStart();
        } catch (RuntimeException ex) {
            throw new RuntimeException("Failed to start service " + service.getClass().getName()
                    + ": onStart threw an exception", ex);
        }
        warnIfTooLong(System.currentTimeMillis() - time, service, "onStart");
    }
```

![image-20240312093713782](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403120943715.png)



在注释1处将PowerManagerService添加到mServices中，其中mServices是一个存储SystemService类型的ArrayList，这样就完成了PowerManagerService的注册工作。在注释2处调用PowerManagerService的onStart函数完成启动PowerManagerService。

除了用mSystemServiceManager的**startService**函数来启动系统服务外，也可以通过如下形式来启动系统服务，以PackageManagerService为例：

![image-20240312093935812](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403120943717.png)

在注释1处直接创建PackageManagerService 并在

注释2处将PackageManagerService注册到ServiceManager中，ServiceManager用来管理系统中的各种Service，用于系统C/S架构中的Binder通信机制：

**Client端要使用某个Service，则需要先到ServiceManager查询Service的相关信息，然后根据Service的相关信息与Service所在的Server进程建立通信通路，这样Client端就可以使用Service了。**

## 总结

（1）启动Binder线程池，这样就可以与其他进程进行通信。

（2）创建SystemServiceManager，其用于对系统的服务进行创建、启动和生命周期管理。

（3）启动各种系统服务。























































