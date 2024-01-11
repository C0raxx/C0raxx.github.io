---
layout:     post
title:      【NSSCTF刷题】《Easy SMC》
subtitle:   CTF
date:       2024-01-10
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【CTF】
    - 【逆向】
    - 【NSSCTF】
---

tmd终于快把nssctf的re题刷完了

![image-20240111212337398](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401112132379.png)

# 题目

![image-20240111212255890](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401112132381.png)

## 解法

其实挺搞的 我自己做的时候根本没有做smc相关的东西

先exe看一下，无壳64elf

![image-20240111212459803](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401112132382.png)

反编译看了一下蛮简单的。

![image-20240111212527260](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401112132383.png)

但是有个问题，其实这里是个待解码的smc片段，这里的getnextnumber由于ida的反编译方式，变成了一个简单的加法，算是一个小坑吧。

然后这题一个可写的就是ida调试linux了，目前还比较少，我这次忘掉了，是看了这位师傅的教程会的

https://www.cnblogs.com/DorinXL/p/12732721.html



然后就开始动调，在if，这个地方下断，进汇编看

![image-20240111212813543](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401112132384.png)

发现网上跳到这里，换到这个地方下断点

![image-20240111212855172](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401112132385.png)

然后我发现。。这里的这个判断条件，把flag明文直接给edx，输入的给eax，直接dump下来就是flag

`NSSCTF{h35Y0viBlAvL5_1nt3re5ting_s31f_m0d1fy_c0d3_18WSHDV3u7oCT}`

![image-20240111212942436](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401112132386.png)



然后应该算是标准方法把，就是进汇编里的getnextnumber看4

（这里已经标记分析过了，正常这边是一堆dd，c然后创建函数f5，出伪代码

![image-20240111213039365](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401112132387.png)

得到一堆形如密文的数组数据，dump下来写脚本

![image-20240111213120157](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401112132388.png)

```
enc = [78, 82, 81, 64, 80, 65, 117, 97, 43, 44, 79, 37, 106, 92, 52, 93, 49, 101, 58, 34, 75, 28, 88, 93, 27, 89, 75, 26, 88, 76, 80, 72, 63, 82, 17, 14, 66, 58, 71, 9, 60, 8, 60, 78, 51, 54, 2, 53, 3, 46, -1, 5, 35, 30, 18, 13, 30, -6, 59, -4, 51, 6, 22, 62]
for i in range(len(enc)):
    print(chr(enc[i] + i), end='')
```

出`NSSCTF{h35Y0viBlAvL5_1nt3re5ting_s31f_m0d1fy_c0d3_18WSHDV3u7oCT}`