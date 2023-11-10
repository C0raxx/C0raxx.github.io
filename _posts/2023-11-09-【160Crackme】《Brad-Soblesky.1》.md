---
layout:     post
title:      【160Crackme】《Brad Soblesky.1》
subtitle:   Crackme
date:       2023-11-09
author:     Corax
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 【Crackme】
    - 【逆向】
---

# Brad Soblesky.1

直接exeinfo

![image-20231110204742178](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311102101869.png)

没加壳，老VC写的，直接OD

一样的查找字符串

![image-20231110204946626](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311102101870.png)

找到这个地方，发现有个关键跳会跳开 回显正确的代码

![image-20231110205034282](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311102101871.png)

先爆破它，直接nop掉jnz

![image-20231110205119062](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311102101872.png)

成功

![image-20231110205124691](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311102101873.png)

再往上分析代码，下断点在cmp之前

![image-20231110205232310](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311102101874.png)

运行，因为前一句就是lea ecx，观察ecx

![image-20231110205253935](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311102101875.png)

得知就是我们输入除了第一个字符，和这个东西比较，直接复制

![image-20231110205317425](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311102101876.png)
ECX 0019F4F0 ASCII < BrD-SoB>
成功。

![image-20231110205434489](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311102101877.png)

