---
layout:     post
title:      【攻防世界逆向】《Shuffle》《666》《dmd-50》《easyRE1》《IgniteMe》
subtitle:   Crackme
date:       2023-09-30
author:     Corax
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 【CTF】
    - 【逆向】
    - 【攻防世界】
---

#题目Shuffle
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113198.png)
##解法
先exeinfo
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113199.png)
32位无壳elf直接放到ida
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113200.png)
可以看到红圈内就是随机处理，那上面的是原始字符串，通过py写个脚本
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113201.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113202.png)
得到flag
#题目666
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113203.png)
毛都没
##解法
先exeinfo
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113204.png)
32位无壳elf 放进ida
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113205.png)
（天真的我直接把下面那个字符串当flag）
可以看到上面那个判断函数是把什么东西跟一个字符串比较了
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113206.png)
所以可以想到，是把我们输入的字符串作加密处理变成这个样子了。看上面的函数，锁定这个
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113207.png)
一个加密过程，逆着写
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113208.png)
unctf{b66_6b6_66b}
#题目Reversing-x64Elf-100
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113209.png)
##解法
这道题蛮简单的
64位无壳elf
扔进ida![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113210.png)
就这里很关键
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113211.png)
依葫芦画瓢
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113212.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113213.png)
#题目dmd-50
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113214.png)
##解法
无语。。
ida打开
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113215.png)
判断过程老长一段，把这些数字换成asc2码 780438d5b6e29db0898bc4f0225935c0
看到上面这里
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113216.png)
md5加解密找个网站
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113217.png)
得到flag
#题目easyRE1
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113218.png)
##解法
不说了看图吧
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113219.png)
记得加上{}
#题目IgniteMe
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113220.png)
##解法
终于来了一个exe。先用exeinfo
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113222.png)
ok直接扔ida
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113223.png)
刚开始看的时候其实蛮懵逼的，细看之后其实很多都是类似输出函数这种很基本简单的函数，主要在于这里
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113224.png)
点进去
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113225.png)
看最后一段，大概是用输入的flag进行一些运算然后和最后面地字符串进行对比，所以分析一下算法把
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113226.png)
这两段是大小写的转换，运算后的flag还要进行大小写的转换。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113227.png)
主要在这个。
是最后的字符串是byte_4420B0[i]和处理过的输入字符串进行异或得到。所以逆向算法。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113228.png)
转换大小写。最后加上这里的外壳![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020113229.png)

就可以了。
