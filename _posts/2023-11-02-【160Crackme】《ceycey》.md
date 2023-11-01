---
layout:     post
title:      【160Crackme】《ceycey》
subtitle:   Crackme
date:       2023-09-30
author:     Corax
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 【Crackme】
    - 【逆向】
---

这个程序其实没什么好写的，但是可以用来做一下壳入门的尝试，又作为一次比较标准的脱upx壳的经验，还是可以看看。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020102688.png)
exeinfo直接爆了标准的upx壳。
这种标准的upx壳，可以直接采用最粗暴的方法，直接upx -d
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020102689.png)
就直接出来
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020102690.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020102691.png)

。
但作为标准的upx，手动脱脱。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020102692.png)
来到程序内部，明显的push 堆栈平衡操作
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020102693.png)
执行之后，注意到ESP和EIP为红色，这里就直接在ESP上面下一个HWbreak，硬件断点
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020102694.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020102695.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020102696.png)
来到这，是OEP的位置，但还是看到很多乱码，没事，分析里面删除模块分析
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020102697.png)
就出来了解压后的程序
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020102698.png)
，本来这里用插件可以直接dump
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020102699.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020102700.png)
。
但脱出来的东西有些问题
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020102701.png)
查了一些资料，说是因为在win7及其之上用了ASLR（这后面细讲），导致出现问题，直接打开xp虚拟机，一样的步骤
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020102702.png)
就可以出来一个可以用的脱壳的程序。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020102703.png)
在win10里面也是正常的。
分析也是正常流程，看一下字符串，找到
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020102704.png)
这里应该是正确的，跟进
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020102705.png)
来到这里，如果正确的话这个jnz不会跳，所以在jnz下断点，
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020102706.png)
可以看到将要实现，我们nop掉
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020102707.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020102708.png)
出现，爆破成功。
算法也没什么逆的，明文存储的。
