---
layout:     post
title:      【NSSCTF逆向】《enc》《easyenc》
subtitle:   CTF
date:       2023-09-30
author:     Corax
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 【CTF】
    - 【逆向】
    - 【NSSCTF】
---

#总览
enc tea rc4 smc加密 
easyenc int逐位转char
#题目enc
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135951.png)
##解法
感觉还是挺难得一个题目
32位无壳 直接扔进ida
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135952.png)
直接找关键字符串
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135953.png)
首先注意到的是right，但是很奇怪没有函数引用，
先不急 先看下面的那个函数 进去
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135954.png)
找到main函数了
看起来是一个很简单的程序
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135955.png)
先来看前半部分
把我们输入的v11，分别给了v10和v7
然后有一个函数对v7进行操作
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135956.png)
详细的操作如上图，很明显是一个tea，ok，退出来
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135957.png)
下半部分有两个函数有看点的，第一个传入了一个v10，详细跟踪一下
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135958.png)
到这里还没完，继续下去
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135959.png)
一个加密方法。而上面是一个典型的smc局部加密代码，看看另外一个函数
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135960.png)
ida报错了，点进去地址看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135961.png)
显然就是被加密的部分了，不着急解密看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135962.png)
这个解密代码之前有做过，拿来用吧。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135963.png)
变成现在这样，还没结束，先按c对他们重新分析一下
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135964.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135965.png)
已经ok了，但是还没有承认这个函数，通过edit function create function来创建函数
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135966.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135967.png)
f5
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135968.png)
有了，是一个rc4。
关于详细的脚本我没写，看了官方的wp https://www.cnblogs.com/Tree-24/p/17346919.html#enc
flag NSSCTF{y0u_ar3_rc4_t3a_smc_m4ster!!}
#题目easyenc
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135969.png)
##解法
感觉这题也是挺抽象的。
无壳64位，丢进ida
进入后直接反编译
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135971.png)
看上去还是蛮简单的，来分析一下
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135972.png)
一些赋值，框中内容是一个输入函数
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135973.png)
v4这里是用一个do while然后对输入字符串的长度进行一个查询。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135974.png)
这里是关键了，v4应该是41，也就是输入的应该是41位
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135975.png)
这两条是最抽象的。
实际上是逐位读v10，而v10是一个4个字节的int，这里操作实际上是取1个字节 等于是取一个字符。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135976.png)
然后这个字符和v8的一位比较，对的话就正确。
而v8就在上面。
所以逆向思路就是通过这个v8来拼。
我自己做的时候是手动的，极其麻烦，这里贴一下别的师傅的写法
https://blog.csdn.net/qq_46266259/article/details/128642972
flag![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135977.png)
