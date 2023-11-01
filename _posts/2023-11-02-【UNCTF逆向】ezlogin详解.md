---
layout:     post
title:      【UNCTF逆向】ezlogin详解
subtitle:   CTF
date:       2023-09-30
author:     Corax
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 【CTF】
    - 【逆向】
    - 【UNCTF】
---

#题目ezlogin
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121814.png)
##解法
###1.解法一
打开程序稀里糊涂先打开
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121815.png)
注册都不用注册，直接选择login
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121816.png)
用户名密码选择1
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121817.png)
就成功了，但感觉应该是程序哪里有问题，还是按照题目要求的做一遍。
###2.解法二
先用exeinfope打开![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121818.png)
没有加壳，用ida打开。
程序流程还是比较清晰的，很容易就找到了login函数。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121819.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121820.png)
点进去再按f5看到伪代码。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121821.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121822.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121824.png)
锁定到最后那行调用到了messageboxa的api函数，这里应该就是关键。
按回汇编代码可以看到显示登陆成功的地方前面有个跳转
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121825.png)
往上看发现这里有个jge，应该是个关键跳转。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121826.png)
分析得差不多了，想运用动态调试看看。
用Ollydbg打开后设置了几个断点，让我把jge直接改成了jmp。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121827.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121828.png)
可以看到纵使我随便乱输也成功得到了flag。
##总结
第二题做的比之前那道轻松些了，但也有不少的阻碍，虽然找到了login的函数代码，但苦于不善将伪代码中的加密flag复现出来，所以不得不去学习如何使用动态调试，本来想用ida进行动态调试的，但是又有各种问题无法进行，转头奔向ODG，不太会用就现场在bilbili简单学了一下如何使用。
