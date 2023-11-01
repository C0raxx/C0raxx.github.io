---
layout:     post
title:      【攻防世界逆向】【高手题】《re-for-50-plz-50》《srm-50》《Mysterious》《Guess-the-Number》《answer_to_everything》
subtitle:   Crackme
date:       2023-09-30
author:     Corax
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 【CTF】
    - 【逆向】
    - 【攻防世界】
---

#题目re-for-50-plz-50
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111186.png)
##解法
题目不难，先exeinfo![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111188.png)
32位elf无壳，但是我在做的时候碰到了一些困难，原本用的是低版本的ida，在f5进行反汇编的时候失败了，然后在吾爱下了一个新版本的ida，就反汇编成功了。以下
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111189.png)
看起来非常简单明了，关键在于有一个字符串和55进行了异或，点进去看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111190.png)
这样一个，好编写脚本
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111191.png)
得到flag![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111192.png)
#题目srm-50
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111193.png)
##解法
解出这题十分开心
给了一个程序 让我们数邮箱和编号
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111194.png)
用exeinfo打开看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111195.png)
ok扔进ida32因为是win程序 所以是winmain窗口
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111196.png)
反汇编![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111197.png)
逐一点开这个函数是关键
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111198.png)
有一个很长的判断过程
不用看那些很花哨的，找成功的字符串
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111199.png)
可以看到是吧success复制到一个字符串，通过一个判断把失败或成功赋值给字符串，再输出。所以直接编写脚本
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111200.png)
ok
#题目Mysterious
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111201.png)
##解法
这道题也很有意思，特别是看了别人的wp后。首先还是先打开看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111202.png)
输入窗口，想不出来，exeinfo打开看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111203.png)
32位无壳，扔ida里，跟上题一样是winmain函数名，一步步跟进来到这里（也可以字符串定位）
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111204.png)
可以看到中间的一部分是成功后输出的，是flag，都很全了，但是缺少一个量v5，v5在这条里面被赋值![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111205.png)
打开里面看一看这是一个字符串数字转换的一个函数，那上面具体对v10做了什么呢？
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111206.png)
也可以看到这里v10是先字符串转数字的，下面甚至直接给到v10的值，为123
加入下面的一行就是flag{123_Buff3r_0v3rf|0w}
看了别人的wp，没想到这题其实还涉及了一些关于溢出的一些知识。我把我学到的简要写一些。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111207.png)
这里输入的时候没有对string转入的字符串作长度验证，在if中做了，意思要小于等于6位
再往下看v10是转成数字之后加了1，那么这个v10输入的部分前面应该是122，重点在于下面的判断条件
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111208.png)
v12v14v13为何？借用动态调试可以看到从12 13 14是xyz
那可以看到上面没对输入字符串作验证，那么123后面的是什么呢 是他内存地址连在一块，巧用溢出的xyz，所以输入内容应该是122xyz
#题目Guess-the-Number
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111209.png)
##解法
这道题下载下来是一个jar文件吧，从来没见过这样的，就直接打卡wp去看了。
他其实可以解压，可以解压获得一个class文件，根据学习class是文件是.java变成字节码后的文件，需要用到java的反编译才能看到代码。这里有两个方式都成功了。一个是idea可以打开class![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111210.png)
另外一个是jdgui
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111211.png)
也可以看到的。
那么接下里就是看里面的内容，可以看到他是两个数进行异或然后输出。很简单我们直接编写脚本
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111212.png)
获得答案。
#题目answer_to_everything
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111213.png)
##解法
这道题的介绍还是挺好玩的
下载下来是一个zip，解压获得一个exe。打不开，用exeinfo看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111214.png)
扔进ida64
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111215.png)
函数非常简单，看nottheflag
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111216.png)
很了然，上面的那个submit without tags，所以只要kdudpeh就可以了
但是提交上去显示错误。
猛然发现介绍中提到了sha1，去了解了一下这个算法，还缺一个值，到底在哪里？
无奈翻看别人的wp去了，学到了新的知识。
在cmd当中powshell
然后有两串命令
certutil -hashfile .\CentOS_7.rar MD5
certutil -hashfile .\CentOS_7.rar SHA1
可以看到md5或者sha1值
如下
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111217.png)
复制下来，找个在线网站
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111218.png)
解得
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111219.png)
填入flag，成功。