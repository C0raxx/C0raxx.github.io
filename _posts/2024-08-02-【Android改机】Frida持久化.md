---
layout:     post
title:      【Android改机】Frida持久化
subtitle:   Android改机
date:       2024-08-02
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【Android改机】
    - 【逆向】
    - 【Android】
---

# Frida持久化

这一块的内容最好先源码导入到AS里面去，这个要生成一个idegen.jar文件再导入，可以百度一下。

本篇文章主要是记录一下，我在**将Frida-gadget持久化方案通过改机进行实现的一个过程和步骤**。

不管是我们之前用的server，还是这个gadget，其本意都是**完成注入，注入一个能实现所需要hook功能的东西**，注入到目标进程里面。

省流资源分享

```
编译成果
https://github.com/C0raxx/AndroidCompile/tree/main/Android10/Android10Frida
```



## 1.前置

```
frida-server
	是比较普遍使用的方法，网上教程都这么教，也比较简单，问题是需要root和容易被检测。
重打包
	对apk解包后修改samli植入我们的frida-gadget（或者patch so对原lib进行操作），再重打包，达成hook，也无需ptrace。问题是需要进行签名的校验，这个得看目标app是啥了。
编译系统
	自己编译系统，修改系统源码，将frida-gadget集成到系统中，使得app在启动时首先动态加载frida-gadget。我们其实能直接修改系统源码来影响app的动态执行过程，且不引入frida特征，但是集成frida的优势在于能够hook app自身的代码，且修改hook点较为方便。
```

这篇的主题就是第三种的尝试，**使其app启动过程中，自动加载frida-gadget**。

## 2.判断是否启用持久化

主要是在frameworks/base/core/java/android/app/ActivityThread.java里面的handleBindApplication里面

选择一个时机，不要太晚就行，我选在这后面

![image-20240807174250836](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080713043.png)

添加上后面这段代码

```java

// add
        String curPkgName = data.appInfo.packageName;
        int curUid = Process.myUid();
        if (curUid > 10000) {
            Persist.LOGD("curPkgName: " + curPkgName + " curUid: " + curUid);
            Boolean isPersist =  .isEnablePersist(curPkgName);
            Persist.LOGD("isPersist: " + isPersist);
            if (isPersist) {
                if(Persist.doXiaojianbangPersist(appContext, curPkgName)){
                    Persist.LOGD("doXiaojianbangPersist is ok");
                }else {
                    Persist.LOGD("doXiaojianbangPersist failed");
                };
            }
        }
// add

```

可以看到主要功能是

1. 先判断一下这个进程是不是我们的用户进程，如果是系统的就没必要进行加载。
2. 如果是用户进程，会先调用一个函数Persist.isEnablePersis来判断有没有一个标志文件（后面做持久化管理的时候用到的，如果打开了持久化开关，就会在特定目录释放一个文件被这个isEnablePersis检测到）关于这个函数我们后面继续看。
3. 然后根据isEnablePersis得到的isPersist的值，进行doXiaojianbangPersist的调用，这个里面就是把我们frida gadget载入的地方了。

现在这两函数不知道干啥的，接着看。

## 3.增加自定义包和类

在这里我们新建一个类和文件

![image-20240807175321029](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080713044.png)

里面copy进我们的代码

```java
package com.xiaojianbang;

import android.content.Context;
import android.util.Log;
import android.os.Process;

import java.io.File;
import java.io.FileInputStream;
import java.io.FileOutputStream;

import org.json.JSONObject;


public class Persist {

    public static final String SO_NAME = "libxiaojianbang.so";
    public static final String SO_CONFIG_NAME = "libxiaojianbang.config.so";
    public static final String LIB32_DIR = "/system/lib";
    public static final String LIB64_DIR = "/system/lib64";

    public static final String SETTINGS_DIR = "/data/system/xsettings/xiaojianbang/persist";
    public static final String ENABLE_PERSIST_FILE_NAME = "xiaojianbang_persist";

    public static final String CONFIG_JS_DIR = "/data/system/xsettings/xiaojianbang/jscfg";
    public static final String CONFIG_JS_FILE_NAME = "config.js";

    public static final String TAG_NAME = "xiaojianbang_persist";


    public static void LOGD(String msg) {
        Log.d(TAG_NAME, msg);
    }

    private static boolean saveFile(String filePath, String textMsg) {
        try{
            FileOutputStream fileOutputStream = new FileOutputStream(filePath);
            fileOutputStream.write(textMsg.getBytes("utf-8"));
            fileOutputStream.flush();
            fileOutputStream.close();
            return true;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return false;
    }

    private static boolean copyFile(File srcFile, File dstFile) {
        try{
            FileInputStream fileInputStream = new FileInputStream(srcFile);
            FileOutputStream fileOutputStream = new FileOutputStream(dstFile);
            byte[] data = new byte[16 * 1024];
            int len = -1;
            while((len = fileInputStream.read(data)) != -1) {
                fileOutputStream.write(data,0, len);
                fileOutputStream.flush();
            }
            fileInputStream.close();
            fileOutputStream.close();
            return true;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return false;
    }
    // 判断app是否打开自动注入脚本功能
    public static boolean isEnablePersist(String pkgName) {
        // 判断文件是否存在 /data/system/xsettings/xiaojianbang/persist/com.xiaojianbang.app/xiaojianbang_persist
        File enableFile = new File(SETTINGS_DIR, pkgName + File.separator + ENABLE_PERSIST_FILE_NAME);
        return enableFile.exists();
    }
    // 获取源JS文件路径
    private static File getConfigJSPath(String pkgName) {
        // /data/system/xsettings/xiaojianbang/jscfg/com.xiaojianbang.app/config.js
        return new File(CONFIG_JS_DIR, pkgName + File.separator + CONFIG_JS_FILE_NAME);
    }
    // 拷贝源JS文件到app私有目录
    private static File copyJSFile(Context context, String pkgName) {
        // 判断源JS文件是否存在
        File srcJSFile = getConfigJSPath(pkgName);
        if(!srcJSFile.exists()) {
            LOGD("srcJSFile not exists");
            return null;
        }
        // 拷贝源JS文件到app私有目录
        // /data/data/com.xiaojianbang.app/files/config.js
        File dstJSFile = new File(context.getFilesDir(), CONFIG_JS_FILE_NAME);
        boolean isCopyJSOk = copyFile(srcJSFile, dstJSFile);
        if(!isCopyJSOk){
            LOGD("copyJSFile fail: " + srcJSFile + " -> " + dstJSFile);
            return null;
        }
        return dstJSFile;
    }
    // 生成Gadget配置文件
    private static boolean genGadgetConfig(Context context, File dstJSFile) {
        JSONObject jsonObject = new JSONObject();
        JSONObject childObj = new JSONObject();
        try {
            childObj.put("type", "script");
            childObj.put("path", dstJSFile.toString());
            jsonObject.put("interaction", childObj);
        }catch (Exception e){
            e.printStackTrace();
            return false;
        }
        String configFilePath = context.getFilesDir() + File.separator + SO_CONFIG_NAME;
        boolean isSaveOk = saveFile(configFilePath, jsonObject.toString());
        if(!isSaveOk){
            LOGD("saveFile fail: " + configFilePath);
            return false;
        }
        return true;
    }
    // 拷贝源so文件到app私有目录
    private static File copySoFile(Context context) {
        // 判断源so文件是否存在
        // /system/lib/libxiaojianbang.so
        // /system/lib64/libxiaojianbang.so
        File srcSoFile = new File(LIB32_DIR, SO_NAME);
        if(Process.is64Bit()) {
            srcSoFile = new File(LIB64_DIR, SO_NAME);
        }
        if(!srcSoFile.exists()) {
            LOGD("srcSoFile not exists");
            return null;
        }
        // 拷贝源so文件到app私有目录
        // /data/data/com.xiaojianbang.app/files/libxiaojianbang.so
        File dstSoFile = new File(context.getFilesDir(), SO_NAME);
        if(srcSoFile.length() != dstSoFile.length()) {
            boolean isCopyFileOk = copyFile(srcSoFile, dstSoFile);
            if(!isCopyFileOk){
                LOGD("copySoFile fail: " + srcSoFile + " -> " + dstSoFile);
                return null;
            }
        }
        return dstSoFile;
    }
    // 进行Frida Gadget持久化
    public static boolean doXiaojianbangPersist(Context context, String pkgName) {
        File dstJSFile = copyJSFile(context, pkgName);
        if(null == dstJSFile) return false;
        if(!genGadgetConfig(context, dstJSFile)) return false;
        File dstSoFile = copySoFile(context);
        if(null == dstSoFile) return false;
        System.load(dstSoFile.toString());
        return true;
    }

}
```

我们研究一下，上回说到俩函数isEnablePersist

过程比较简单，就是**判断某目录下有没有所需要的文件**（这个文件我们后续释放）

![image-20240807181444284](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080713045.png)

还有一个doXiaojianbangPersist

也比较好分析

里面主要涉及到三个问题的操作

![image-20240807183150085](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080713046.png)

对于这三个文件，可以提一下

```
copyJSFile 这个就是正常进行hook的时候的JS文件 继续跟的话可以看到它是调用了一个getConfigJSPath方法把指定目录下的JS文件获得，之后再以这个JS拷贝到app的私有目录

genGadgetConfig 这个是我们frida-gadget必要的东西，frida-gadget要成功发挥作用需要两个东西，一个是frida-gadget-14.2.18-android-arm64.so，另一个是JSON文件，格式如下图。而genGadgetConfig是进行动态地生成这个JSON配置文件，就是得到下面这个格式地JSON。

copySoFile 这个就是拷贝我们的frida-gadget-14.2.18-android-arm64.so到app私有目录下了。然后赋给dstSoFile
```

![image-20240807183835295](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080713047.png)

到这里，如果我们开启持久化，**所需要的一个JS，一个JSON，一个So就有了**，接下来就是载入。

![image-20240807184102803](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080713049.png)

到这里代码层面地工作就完成了，后面需要一些源码编译时候让它把我们添加的包等东西添加编译的工作。

## 3.包加入白名单

/build/make/core/tasks/check_boot_jars/package_whitelist.txt

末尾加入

```
# // add
com\.xiaojianbang
# // add
```

## 4.frida-gadget集成到系统

我们的so文件还没放进去了。

/frameworks/base/cmds/libxiaojianbang下面拷贝

![image-20240807184400146](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080713050.png)

再修改一下mk文件，让它编译的时候把上面这俩so编译进去

/build/make/target/product/handheld_system.mk

```
# // add
PRODUCT_COPY_FILES += \
    frameworks/base/cmds/xiaojianbang/frida-gadget-14.2.18-android-arm.so:$(TARGET_COPY_OUT_SYSTEM)/lib/libxiaojianbang.so \
    frameworks/base/cmds/xiaojianbang/frida-gadget-14.2.18-android-arm64.so:$(TARGET_COPY_OUT_SYSTEM)/lib64/libxiaojianbang.so
# // add
```

## 5.自定义目录设置

用来标记我们是否开启持久化的有一个文件

/data/system/xsettings/xiaojianbang/persist/包名/xxxx

用frida hook的js也有存放的位置

/data/system/xsettings/xiaojianbang/jscfg/包名/config.js

每次开机都要创建太麻烦，比如直接再编译的时候让他**创建好这个目录**。



/system/core/rootdir/init.rc 文件中添加以下数据

```
    # // add
    # /data/system/xsettings/xiaojianbang/persist
    mkdir /data/system/xsettings 0775 system system
    mkdir /data/system/xsettings/xiaojianbang 0775 system system
    mkdir /data/system/xsettings/xiaojianbang/persist 0775 system system
    mkdir /data/system/xsettings/xiaojianbang/jscfg 0775 system system
    # // add

```

## 6.配置system权限

后续要进行hook的时候，是用一个app进行hook管理的，这个app再进行js文件的加载的时候，如果不经过权限配置，只能访问特定目录，所以我们需要配置一下权限。

```
在如下文件中添加数据
/system/sepolicy/private/system_app.te
/system/sepolicy/prebuilts/api/29.0/private/system_app.te
在以上文件中添加如下数据，两个文件添加的内容需要一致 末尾
# // add
# add for accessing xiaojianbang_file
allow system_app xiaojianbang_file:dir  { getattr setattr open read write remove_name create add_name search rmdir };
allow system_app xiaojianbang_file:file { getattr setattr open read write create unlink };

```

## 7.创建文件类型SeLinux标签，文件类型关联

**文件类型的处理**

```
在如下文件中添加数据
/system/sepolicy/public/file.te
/system/sepolicy/prebuilts/api/29.0/public/file.te
在以上文件中添加如下数据，两个文件添加的内容需要一致
# // add
# /data/system/xsettings/xiaojianbang/persist
type xiaojianbang_file, file_type, data_file_type, core_data_file_type, mlstrustedobject;
# // add
```



```
在如下文件中添加数据
/system/sepolicy/private/file_contexts
/system/sepolicy/prebuilts/api/29.0/private/file_contexts

在以上文件中添加如下数据，两个文件添加的内容需要一致 122
  注意文件不要以注释结尾
# // add
# /data/system/xsettings/xiaojianbang/persist
/data/system/xsettings(/.*)?	u:object_r:xiaojianbang_file:s0
# // add
```

## 8.配置第三方app访问 xiaojianbang_file 标签文件的权限

继第六步

```
在如下文件中添加数据
/system/sepolicy/private/untrusted_app.te
/system/sepolicy/private/untrusted_app_25.te
/system/sepolicy/private/untrusted_app_27.te
/system/sepolicy/private/untrusted_app_all.te

/system/sepolicy/prebuilts/api/29.0/private/untrusted_app.te
/system/sepolicy/prebuilts/api/29.0/private/untrusted_app_25.te
/system/sepolicy/prebuilts/api/29.0/private/untrusted_app_27.te
/system/sepolicy/prebuilts/api/29.0/private/untrusted_app_all.te
在以上文件中添加如下数据，两个文件添加的内容需要一致 末尾
# // add
# add for accessing xiaojianbang_file
allow untrusted_app xiaojianbang_file:dir  { getattr open read write search rmdir };
allow untrusted_app xiaojianbang_file:file { getattr open read write };
```



```
/system/sepolicy/private/compat/26.0/26.0.ignore.cil 17
/system/sepolicy/private/compat/27.0/27.0.ignore.cil 16
/system/sepolicy/private/compat/28.0/28.0.ignore.cil 15

/system/sepolicy/prebuilts/api/29.0/private/compat/26.0/26.0.ignore.cil 17
/system/sepolicy/prebuilts/api/29.0/private/compat/27.0/27.0.ignore.cil 16
/system/sepolicy/prebuilts/api/29.0/private/compat/28.0/28.0.ignore.cil 15

在以上文件中加入数据 xiaojianbang_file
两个文件添加的内容需要一致
```

## 9.App

因为我看的就是xjb的课程，直接使用它的工程了。

这个我直接集成进源码了

```
管理app的功能
    显示已安装app列表
    可以对每个app指定需要注入的JS
    可以设置是否启用持久化
```

1.2 将编译出来的app放入 /packages/apps/xiaojianbangPersist
1.3 编写Android.mk，也放入该文件夹
1.4 单独编译指定模块 mmm packages/apps/xiaojianbangPersist
1.5 编译后的模块在 /out/target/product/sailfish/system/app/ControlAPP
1.6 使用 make snod 将编译出来的文件打包成镜像，刷入system.img即可



在这个/build/make/target/product/handheld_product.mk

```
# // add
# 设置当前工作路径
LOCAL_PATH:= $(call my-dir)


# 清除变量值
include $(CLEAR_VARS)
# 生成的模块名称
LOCAL_MODULE := ControlAPP


# 生成的模块类型
LOCAL_MODULE_CLASS := APPS
# 生成的模块后缀名,此处为apk
LOCAL_MODULE_SUFFIX := $(COMMON_ANDROID_PACKAGE_SUFFIX)
# 设置模块tag，tags取值可以为:user debug eng tests optional
# optional表示全平台编译
LOCAL_MODULE_TAGS := optional

# LOCAL_PRIVILEGED_MODULE := true

LOCAL_BUILT_MODULE_STEM := package.apk

LOCAL_DEX_PREOPT := false

# 设置源文件
LOCAL_SRC_FILES := $(LOCAL_MODULE).apk

LOCAL_CERTIFICATE := platform

# 设置签名，此处表示保持apk原有签名
# LOCAL_CERTIFICATE := PRESIGNED
# 此处表示预编译方式
include $(BUILD_PREBUILT)
```

# 总结

到这边就差不多了，成功编译的结果放在了开头。

这篇里面我们主要是动手操作，**修改了应用的启动流程**，让他在启动的时候就加载了gadget和他的配置文件，以及拷贝好hook用的js文件。然后配合管理的App对系统做了一些修改