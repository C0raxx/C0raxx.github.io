---
layout:     post
title:      【UNCTF逆向】Rejunk详解
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

进行了一学期的纯理论学习，深感实战的重要性，而在现阶段没有什么项目可以实操，故先从CTF题目开始做起，首先先熟悉熟悉各个工具的使用和逆向思路。

#题目Rejunk
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020120875.png)
是一道从垃圾代码中找到价值代码，进行异或算法识别找到flag的题目。
##解法
先用exeinfope查壳工具打开看一下文件的情况。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020120876.png)
没有加壳，使用IDA进行打开。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020120877.png)
是一个相当复杂的程序，还是找不到入手点，打开程序看看。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020120878.png)
可以看到这个程序是需要我们输入一串字符，可能对字符进行某种运算（题目上讲明是异或运算），在与程序自带的字符串进行比较。所以在ida当中就可以打开strings。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020120879.png)
可以看到这个程序当中的字符串，经过测试看到我的电脑存储方式是小端法，即高位存在高地址，所以准确的字符顺序应该是从WQG到twk。现在相当于已经找到了加密的字符，现在还不知道题目是进行了哪种异或运算使得这个flag变成被加密字符串进行比较，这个时候应该看看代码了。
按F5将汇编语言反汇编看到了伪代码，可以见得这边。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020120880.png)
Dest这个字符串已经出现。v10这个量在【】当中，应该比较类似于string[i]的i的地位。而v16应该是我们输入的字符串存放的位置。所以异或操作也显而易见了。
就是待增量i和string[i]进行异或再加2得到了WQG..。所以只要反过来，WQG...^i-2就可以了。写一个简单的脚本。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020120881.png)
运行得到flag
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020120882.png)
##总结
因为算是正经做的第一题，所以绕了不少弯路。比如看到垃圾代码不知道哪里下手，搜了一下题解看到可以通过strings找到字符串，比如反过来从‘’twk到GUL，这里也是没搞清大端法和小端法的存储区别。比如扔到ida后查看伪代码显示失败，需要扔到32位（可能因为程序是32位而脱壳工具没显示）。比如不知道别人怎么写出来的key（也就是详细的异或操作)，还重新学了一遍异或操作。比如不知道在char量和int在进行异或操作时是怎么实现的，后面查了是由char先变成asc2值再进行运算的，然后再对应asc2显示成字符变成需要的flag。