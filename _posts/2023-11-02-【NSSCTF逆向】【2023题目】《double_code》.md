---
layout:     post
title:      【NSSCTF逆向】【2023题目】《double_code》
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

#题目double_code
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137410.png)
##解法
解法不是很难，但是没见过。
下载下来是一个exe和一个文本文档
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137412.png)
大概长这样
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137413.png)
应该跟flag有关系，然后看看exe
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137414.png)
直接64位ida打开
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137415.png)
其实还不太好找。
往下拉 是一个很明显的shellcode
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137416.png)
关键在于这个写入进程的这个函数
点进去
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137417.png)
其实看逻辑也可以看出来
但是我看wp，这里涉及到一个shellcode虚拟机的写法，然后ida识别的会比较抽象
按照wp的做法 先找到这段在hex存的地址
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137418.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137419.png)
定位到这一长串，然后另存为一个文档
wp上写用010editor打开，我下载下来了，
但是目前还是没看到这玩意跟普通的hexview有啥区别
没事接着做，用ida32打开
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137420.png)
长这样，还没结束，需要创建函数然后f5
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137421.png)
这样子逻辑就很明显了，可以编写exp
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137422.png)
flag HDCTF{Sh3llC0de_and_0pcode_al1_e3sy}
