---
layout:     post
title:      【攻防世界逆向】open-source详解难度3
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

#题目
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020107266.png)
#解法
下载下来是一个C语言源文件
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020107267.png)
直接用vis打开如下
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020107268.png)
可以看到过程并不复杂，并且可以明显见得红框部分就是对flag的计算，然后用16进制进行输出。
那我就想办法能不能跳过判断条件直接获得。
可以看到其中关键点有三个

* first
* second
* strlen（argv【3】）
而first很明显就是0xcafe
second当中有写到atoi这个函数，就是将字符串转换成整型int，但不关键
看他下面判断条件，要一个模5不为3，且模17为8的数，直接选一个最小的数 25
而strlen（argv【3】），在代码中可以看到
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020107269.png)
也就是h4cky0u的长度，为7,。全部填入。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020107270.png)
运行一下
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020107272.png)
得到flag