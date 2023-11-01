---
layout:     post
title:      【NSSCTF逆向】《stream》《a_cup_of_tea》
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

#总览
stream RC4 base64 exe解包 pyc反编译
a_cup_of_tea tea
#题目 stream
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134032.png)
##解法
拿到题目是一个exe，先用exeinfo打开看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134033.png)
可以看到这个程序使用python写的，又是exe。
所以就是常规的思路，exe解包到pyc文件，再对pyc文件进行反编译
先是解包 用pyinstxtractor.py
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134034.png)
里面找到它![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134035.png)
再来反编译他
原本想用uncompyle6的，但是可能是因为用的是高版本，支持不到。
所以用的是pycdc
用powershell打开
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134036.png)
可以看到py源代码了
分析一下这个程序
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134037.png)
流程就是输入一个text，然后程序里面有个key，先将这个key进行rc4加密，然后用加密后的key和text逐位进行一个处理，在进行base64的加密。
由此逆向思路就是先把密文给base64解密，再将这个加密后的key与解密后的密文进行逐位处理（主要是异或，所以可以直接逆）。出来的就是flag。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134038.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134039.png)
得到flag。
##总结
经验

1. pycdc的用法
2. pyinstxtractor的用法
3. rc4直接逆
4. decode 将字符解码（应该是需要配合base64加解密使用）
#题目 a_cup_of_tea
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134040.png)
##解法
这道题目拿到手是一个exe，拿exeinfo看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134041.png)
无壳64位 ida看看
进入主函数![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134042.png)
可以看到框框里面的东西一个就是关键的加密函数了（print，input已经重命名）点开来看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134043.png)
点开来看看，确实是一个tea的加密算法。
结合以前学过的知识，这个tea需要相反过来。那就比较好些。
但是碰到了什么问题估计，这里的v3，v2，v4命名可能有点问题
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134044.png)
看了别人的wp，知道了这里其实是0x12345...，这样的形式。
比较不解的是传入的这个参数取【1】后为啥是这样的。。
还是解题先，有了key，有了buf2的密文，解密就比较容易了，直接写
（还在想为啥是0x12345678，就直接贴wp了）
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134045.png)
##总结
1. tea基本算法
