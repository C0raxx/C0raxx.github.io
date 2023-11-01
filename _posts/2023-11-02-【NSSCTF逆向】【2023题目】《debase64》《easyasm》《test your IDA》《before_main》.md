---
layout:     post
title:      【NSSCTF逆向】【2023题目】《debase64》《easyasm》《test your IDA》《before_main》
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

debase64 变种base64解密
easyasm 简单的汇编题
test your IDA 签到
before_main base64换表

#题目debase64
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020138249.png)
##解法
还是这道题
想明白了前两天没想明白的一个点。
之前主要没搞明白为什么逆序并base64解密呢？
就在于![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020138250.png)
答案就在于这里的传递方法
先传进来的高位 字符放在了低位的地方
而低位放在高位，然后解base64 找到每个字符在base64表中的索引 查出对应的明文。
#题目easyasm![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020138251.png)
##解法
一道很简单的汇编题目
文件下载下来是一个txt文件
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020138252.png)
应该是从ida当中哪个地方扒下来的，里面![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020138253.png)
红框部分大部分都是传参，真正运算的在xor xx，33h，
结合下面的密文 很容易写出exp
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020138254.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020138255.png)
虽然很简单 但是还是可以把汇编知识复习一下
```
eip：运行程序下一个指令的地址
ebp：栈的基地址指针，指向栈底
esp：栈的栈顶指针，指向栈顶
push：入栈，esp 指向地址减 4
move：数据传送指令，完成赋值
jmp：无条件跳转指令，转移到指令指定的地址执行相应的指令
jge：大于或等于转移指令，用于对比内存中两个对象的大小关系
cmp：比较指令，执行从目的操作数中减去源操作数的隐含减法操作，并且不修改任何操作数
EAX：“累加器”(accumulator), 它是很多加法乘法指令的缺省寄存器。
EBX：“基地址”(base)寄存器, 在内存寻址时存放基地址。
ECX：计数器(counter), 是重复(REP)前缀指令和LOOP指令的内定计数器。
EDX：用来放整数除法产生的余数
```
#题目test your IDA
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020138256.png)
##解法
签到题 放到ida就有![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020138257.png)
#题目
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020138258.png)
过程很简洁 重点在于这
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020138259.png)
很明显的base64编码方式
再看看查的是什么表
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020138260.png)
跟一个函数有关
跟进
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020138261.png)
就是这玩意
然后通过网站
http://web.chacuo.net/netbasex
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020138262.png)
出flag
