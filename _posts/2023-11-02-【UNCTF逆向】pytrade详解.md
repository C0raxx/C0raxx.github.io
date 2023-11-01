---
layout:     post
title:      【UNCTF逆向】pytrade详解
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

前段时间有点别的东西在忙，最近会加大力度。
#题目pytrade
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020120966.png)
##解法
这道题的内容是一些opcode也就是python编译的字节码。
网上搜的一些教程是叫手扒，就简单学习了一下。
变量const fast（有形参和局部变量之分）global（全局）
数据结构list dictionary slice
循环while for in if
函数 函数范围（从0开始到returnvalue收尾) 函数调用

然后进行一下手动的还原python代码 主要是中间的循环过程比较关键
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020120967.png)
这里两个callfunction就代表着range（len（flag）），getiter到foriter就是从i开始循环
循环的具体内容就是
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020120968.png)
这里有一条，调用了ord函数，内容就是ord（flag【i】）后面又有加一的操作
所以是ord（flag【i】）+i，但还没结束，后面又有k%3后加一 进行一个异或。
所以这一句就是num[i]=(ord(flag[i]) + i) ^  (k % 3  + 1)
下一条更复杂一些
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020120969.png)
开始还是ord操作，但内容不想上一条那么简单。先把len（flag）绑在了一起，并与i进行一个相减，再与1进行一个相减。至此把len（flag）-i-1放在一块放进flag【】框框里面，flag【】放在ord里面，还没有结束。
后面又调用len（flag）与之相加并减去i减去1，后面同样是k%3加一，进行异或。
存在哪里呢？
先调num，len（flag）减i减一，存到这个里面
所以大概可以得出（别人复现的）
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020120970.png)
出来的是这个[115, 120, 96, 84, 116, 103, 105, 56, 102, 59, 127, 105, 115, 128, 95, 124, 139, 49]
然后做个逆向脚本
就是主要对上面两行代码逆向。
原本的flag【i】=(num[i] ^ (k % 3 + 1)) - i
flag【17-i】=num【17-i】^（k%3+1）+i-17
也就是
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020120971.png)
运行一下得到
py_Trad3_1s_fuNny!
填入正确。
