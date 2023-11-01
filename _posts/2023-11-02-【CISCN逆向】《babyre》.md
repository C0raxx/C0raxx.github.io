---
layout:     post
title:      【CISCN逆向】《babyre》
subtitle:   CTF
date:       2023-09-30
author:     Corax
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 【CTF】
    - 【逆向】
    - 【CISCN】
---

虽然两天就解出来一道题，但是好歹是写了，还是写个wp记录一下吧。
#题目babyre
baby都能做出来的re。。
##解法
其实这道题我也头疼了很久。
因为下载下来是一个xml文件，我没做过类似的逆向题。
打开之后看到是一个xml文档，很乱，虽然捕捉到了类似于“welldone”这种关键的回显信息，但是对xml的整个框架是完全一头雾水。
但我注意到了在文件的开头
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020145737.png)
查到了这个snap好像是一个什么编译环境，我们可以把它下载下来
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020145738.png)
然后打开它
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020145739.png)
可以看到框框中的部分有一个secret的操作，然后在右边
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020145740.png)
是i和i-1 位的一个异或操作。
所以我们，首先要先把这个secret还原出来，通过自己人工把插入的位置都给解决好，解决。
k1=[102,10,13,6,28,74,3,1,3,7,85,0,4,75,20,92,92,8,28,25,81,83,7,28,76,88,9,0,29,73,0,86,4,87,87,82,84,85,4,85,87,30]
然后我们再写算法
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020145741.png)
得到flag
Flag：flag{12307bbf-9e91-4e61-a900-dd26a6d0ea4c（不知道为什么少了个后括号，自己加上去了)