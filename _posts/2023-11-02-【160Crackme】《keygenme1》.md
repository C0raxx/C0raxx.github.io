---
layout:     post
title:      【160Crackme】《keygenme1》
subtitle:   Crackme
date:       2023-09-30
author:     Corax
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 【Crackme】
    - 【逆向】
---

上来先看一下程序信息
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020101437.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020101438.png)
打开页面后输入username和序列号
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020101439.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020101440.png)
check之后报错了。
既然没壳，直接扔od里面。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020101441.png)
定位关键字符串
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020101442.png)
然后追踪到goodboy的地方，可以看到这里有个关键跳，决定这个程序报的是正确还是错误
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020101443.png)
。
试运行一下这个程序，输入错误的信息，然后运行到这个跳
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020101444.png)
可以看到这里的信息是不跳转，我们修改zf标志位
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020101445.png)
成功了，可以看到这个程序的爆破很简单。
。
接下里把算法给逆出来
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020101446.png)
首先是两个很明显的获取输入用户名和序列号
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020101447.png)
这里的这些不用管，是about会出现的信息
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020101448.png)
随后获取字符串的长度，然后和0x4和0x32(也就是50）进行比较，这里用到了jl和jg的内容
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020101449.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020101450.png)
具体这个cmp其实可以理解成一个减法公式，如果结果是负数，则sf符号位记为1，jl根据sf进行决定是否跳转
然后下面遇到的第一个关键点是这里
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020101451.png)
按照顺序给我字符串字符的ascii，然后减去19，然后ebx为上一个ebx减去eax，循环次数为字符串长度
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020101452.png)
出循环之后ebx的值就是ffffff9a
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020101453.png)
经由一个wprintfa打出来
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020101454.png)
然后是第二个字符串的处理
详细的步骤写注释上了。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020101455.png)
出来之后数据室FFEFCEA8
然后是第三个
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020101456.png)
这玩意相对奇怪一点，在40129B上面的处理当中用了之前的ebx，处理到eax，为40e0f8，然后自乘，是有溢出的。
也就是，无论我们输入的值为多少，最后ecx又ebx自乘，必定溢出，低位必定相同，直接看这次的是多少
41720f48。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020101457.png)
拿之前输入的试试，没错。可以写注册机了
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020101458.png)

第一个
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020101459.png)
第二个
```
username="huangyongyong"
key0="Bon"
hjnum=0
eax=0
ebx=0
for i in username:
    eax=ord(i)
    eax=eax-0x19
    ebx=ebx-eax
str1=-ebx-1
str2=hex(str1^0xffffffff+0)
key1=str2.upper()[2:10]
eax=ebx*ebx
ecx=eax
edx=0-ebx
edx=edx^eax
ebx=-ebx*eax-1
str3=hex(ebx^0xffffffff)
key2=str3.upper()[2:10]
key3="41720F48"
key=key0+"-"+key1+"-"+key2+"-"+key3
print(key)

```
测试一下
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020101460.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020101461.png)
成功