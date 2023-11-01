---
layout:     post
title:      【NSSCTF逆向】【2023题目】《kunmusic》
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

#题目 kunmusic
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135436.png)
##解法
这题还是非常有意思的。
打开
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135437.png)
有很多button，可能是需要按button的次数来得到flag把。
这是一个.net的程序，需要用dnspy来反编译他
反编译这个dll
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135438.png)
找到这个入口点
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135439.png)
可以看到是引入了某片数据，然后进行异或104，进行一个解密。
找到这个东西、
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135440.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135441.png)
把他保存下来，然后写一个脚本
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135442.png)
解密出来一个东西
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135443.png)
应该已经是一个可执行文件了 
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135444.png)
进入dnspy反汇编，出来这个东西，用z3写个脚本
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135445.png)
出flag。
这几天会再详细看看z3的用法