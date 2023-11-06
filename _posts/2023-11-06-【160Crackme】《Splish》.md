---
layout:     post
title:      【160Crackme】《Splish》
subtitle:   Crackme
date:       2023-11-06
author:     Corax
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 【Crackme】
    - 【逆向】
---

# Splish

打开看看

分为两个部分，一个是求Hardcoded，一个写name和serial注册机。

![image-20231107014132604](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311070159256.png)

exeinfo查一下信息，无壳，汇编写的。

![image-20231107014246721](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311070159257.png)

## Hardcoded

直接放进OD里看看，先找hardcoded关键处

![image-20231107014356733](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311070159258.png)

往上找，这里上面刚刚调取API获得了输入的文本，下面就比较了，感觉流程并不长，下个断

![image-20231107014505148](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311070159260.png)

看起来像一个比较循环，先把jnznop掉看看能不能爆破

![image-20231107014611302](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311070159261.png)

爆破成功

![image-20231107014624757](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311070159262.png)

看下算法，其实很简单，可以看到eax存了明文的hardcoded，然后一位一位取出来，和我们输入的比较。所以就是“HardCoded”

![image-20231107014703577](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311070159263.png)

成功

![image-20231107014814357](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311070159264.png)

## Name Serial

一样的道理，通过字符串定位到这里

![image-20231107014915851](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311070159265.png)

往上拉，流程有一点长，下断到调用完API

![image-20231107014945091](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311070159266.png)

这里把name字长给eax

![image-20231107015012588](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311070159267.png)

这里是一个具体的判断流程了，这里的

```
cdq
idiv ecx
```

作用就是用ecx去除eax的值，然后商给到eax，余数给edx。

比方说第一次流程取了c asii 99，所以经过运算edx里面是9,这个9和ebx（现在是0，这跟着循环增大）异或，异或后加上2，再模10

每次的流程就只有ebx是根据循环次数往上的。

![image-20231107015025102](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311070159268.png)

写脚本就是如下

```
name = "cxyyy"
b=[]

for i in range(5):
    nYu = ord(name[i]) % 10
    nYu = nYu ^ i
    nYu = nYu + 2
    nYu = nYu % 10
```

name加密的东西有了，看看serial有没有做手脚。

答案是有的

和name的处理方法有相似，大概就是我们输入的东西的ascii，除以10，然后余数直接存进去，比如第一个是1，ascii49，所以就是9进去。

![image-20231107015515928](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311070159269.png)





我们现在要根据name出来的东西写注册机，想法就是根据已经得到的name处理数据如1,3,5,4,7，直接加上10的n倍，这个倍数使得结果全部落在可视字符里面，挑选的n为8，当然后面一百多也可以

![image-20231107015758866](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311070159270.png)

脚本如下

```
name = "cxyyy"
b=[]

for i in range(5):
    nYu = ord(name[i]) % 10
    nYu = nYu ^ i
    nYu = nYu + 2
    nYu = nYu % 10
    b.append(nYu+80)

for i in b:
    print(chr(i),end= '')
```

![image-20231107015854197](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311070159271.png)

成功

![image-20231107015916885](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311070159272.png)