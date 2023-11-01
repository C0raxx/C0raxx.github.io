---
layout:     post
title:      【UNCTF逆向】chall详解
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



 

是第一道做到的里面包含加密算法的题目。
#题目chall
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020122332.png)
##解法
无法正常打开，先用exeinfo看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020122333.png)
好像需要用linux打开，没关系用ida看看。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020122334.png)
主函数好像挺简单的，并且注意到今年的年份为key，可以记一下。
按f5查看一下伪代码
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020122335.png)
有几条函数，逐一检查可以看到最后一条。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020122337.png)
RC4：codemaker
打开加解密的在线网站看看
尝试了一下年份不对，可能是前几年的题目，就往前推
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020122338.png)
是这样一串，以UNCTF{}格式输入，就对了。
##总结
目前为止遇到的第一道带有加密算法的题目，又不是Windows程序，无法打开程序又不清楚加密，卡了很久，是看了网上的关于RC4的原理和别人类似题解才做出来的。途中因为搞不明白加解密的原理，所以在很多在线加解密网站当中搞不懂输入格式和输出格式，想着以后得自己写一个来用用。