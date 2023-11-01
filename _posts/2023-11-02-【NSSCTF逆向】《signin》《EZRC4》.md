---
layout:     post
title:      【NSSCTF逆向】《signin》《EZRC4》
subtitle:   CTF
date:       2023-10-01
author:     Corax
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 【CTF】
    - 【逆向】
    - 【NSSCTF】
---

#题目
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020143097.png)
##解法
打开看一下
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020143098.png)
一个待输入命令行，exeinfo
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020143099.png)
有upx的信息，不过版本号9.96有点可疑，感觉像是修改过的
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020143100.png)
直接upx脱不了，应该和上次做的那个pzthon一样改过upx信息。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020143101.png)
可以看到里面的upx被改成了zvm 改回来
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020143102.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020143103.png)
在下面，这里也被修改过，把ZVM改成UPX，版本放着不改
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020143104.png)
可以看到还是9.96，让upx脱
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020143105.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020143106.png)
成功了，至于为什么，后面开篇技术分析讲。
扔进ida
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020143107.png)
还是十分明显的RC4。找到秘钥是justfortest，然后密文是什么
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020143108.png)
一路追查到这个v18，v18又是上面两个数据
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020143109.png)
注意要逆序存放。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020143110.png)
就可以得到flag了。
#题目
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020143111.png)
##解法
非常简单的一道RC4
ida
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020143112.png)
逻辑都非常清晰了，key和data，就是data要变成个字符串麻烦点

```
nData=[0xEB,0XD,0X61,0X29,0XBF,0X9B,0X5,0X22,0XF3,0X32,0X28,0X97,0XE3,0X86,0X4D,0X2D,0X5A,0X2A,0XA3,0X55,0XAA,0XD5,0XB4,0X6C,0X8B,0X51,0XB1]
hex_string = ''.join(format(num, '02X') for num in nData)
print(hex_string)
```
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020143113.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020143114.png)
flag出
