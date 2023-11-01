---
layout:     post
title:      【NSSCTF逆向】【2023题目】《patchme》
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

前段时间在忙开学和协会的事情，逆向也停了些，去玩了玩pwn，近期再起航

#题目 patchme
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134719.png)
##解法
刚拿到这道题的时候有点蒙，一道逆向题目还让我去补漏洞吗。。。不太明白，反正惯例看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134721.png)
elf文件，放到kali里面看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134722.png)
应该是修补好漏洞之后就会出flag。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134723.png)
进来看到这样的，因为最近学了一段时间pwn，知道gets会存在栈溢出，printf的格式化字符串漏洞，但说实话，我不晓得怎么修。。
就卡出了，然后去网上找wp看。看到一个思路，跟着这个思路做了做
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134724.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134725.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134726.png)
其实在这里面藏了个预处理的函数点进去看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134727.png)
进去之后很明显的有一个protect，网上看了看，是linux当中的一个文件权限保护函数，保护后面箭头所指的区域，然后在后面的区域进行了一个异或，我猜测是把smc当中加密的部分给解密出来。
点进去看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134728.png)
乱七八糟的，用idapython解个密
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134729.png)
解完密再按c重新分析一下
再创建个函数，f5就可以了
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134730.png)
加密过程就出现了，但是还要注意，里面是一个字一个字加密，ida反编译把他变成了数组，才会出现5个元素，却反复46遍的问题
所以脚本就这么写
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134731.png)
但是。。。还是不晓得怎么修啊，看了官方wp，要写在eh_frame，跳转到这个位置，然后再跳出去，途中还要注意寄存器传参
今天有点晚了，明天再看吧