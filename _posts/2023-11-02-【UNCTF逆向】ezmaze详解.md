---
layout:     post
title:      【UNCTF逆向】ezmaze详解
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

#题目ezmaze
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121531.png)
##解法
题目下载下来是一个ezmaze.exe文件，用exeinfo打开看一下
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121532.png)
好像还可以，用ida打开看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121533.png)
刚开始我甚至找不到这个界面，问了一名比较厉害的同学，他告诉我就一个个函数找找看，可能会找到可疑的内容，我就一个个找，最后锁定了这个140001490。打开是这样的
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121534.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121535.png)
反编译一下
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121536.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121537.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121539.png)
有点蒙逼不知道怎么办了，strings界面里面也没有可疑的的字符串，想着就用动态调试的手段试试看吧。
于是乎又去简单学了一下x64dbg（因为好像od没有64位）。
打开x64dbg之后，右击汇编窗口出现搜索，找一下字符串
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121540.png)
刚刚看到这个wwwww跟那个定义1的过程已在一块的，点进去看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121541.png)
果然找到了这些比较关键的代码。
但仍然不知道做了什么，所以上网查了一下，人家的思路是看这个内存窗口，又结合这段代码重置了某些地址的值，想着这不会就是迷宫长什么样吧。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121542.png)
这应该就是了，但还是不晓得是几乘几的啊，就继续找
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121543.png)
这里可以看到是7个一行，所以可以画出来了
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121545.png)
那上下左右怎么做呢，确实又把我难住了，看了别人的wp学了一下，原来是这边![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121546.png)
这是别人的分析![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121547.png)
这样子路线就出来了
DEWEDXDEWWWEDEWEE
试了一下，是这个了。