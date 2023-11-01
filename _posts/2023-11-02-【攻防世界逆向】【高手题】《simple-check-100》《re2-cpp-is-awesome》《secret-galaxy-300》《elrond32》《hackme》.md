---
layout:     post
title:      【攻防世界逆向】【高手题】《simple-check-100》《re2-cpp-is-awesome》《secret-galaxy-300》《elrond32》《hackme》
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

昨天去参加LitCTF了，一共7道re题目，做出来了4道，后面的几道看了一下，不是现在的我能够解出来的，一没思路二没经验。不过也算满意了，这也算是我第一次参加比赛独立做题，加油吧。

#题目simple-check-100
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110205.png)
##解法
这道题我先打开idapro看了一下，![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110207.png)
看起来也不太复杂，不过这种很明显的判断题目，我更想通过动态调试把它解出来，于是我去学习了一下gdb的调试。
先来分析一下这道题的汇编吧。
我们直接锁定![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110208.png)
这里的判断函数应该是重中之重。看汇编代码。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110209.png)
也就是这个地方，jz指令很关键。
要先看test是什么意思
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110210.png)
这是两个不同的值进行比较，所有测试位为0时，令ZF为1
而这里是test eax，eax来判断eax是否为空，只要不为空那ZF必定是0，为空则ZF为1。
jz和je一样，都是相等时跳转（eax为0时跳转），所以这里题目的本来意图是，通过call一个函数让eax为0，从而通过test置zf为1，让je或者jz跳转。
我们要修改的就是不让test成功置zf为0，怎么做？答案就是在它执行前，我们令eax为1就可以了。思路就是这样。
具体操作就是如下
首先gdb 文件，b main在主函数下断点，然后单步执行，到test的地方。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110211.png)
可以看到马上就要执行test了，输入i r eax看一下现在eax的情况
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110212.png)
果然为0，我们通过set $eax=1，再看一下
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110213.png)
ok修改成功，直接c
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110214.png)
成功了！
#题目re2-cpp-is-awesome
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110215.png)
##解法
先用exeinfo打开看看。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110216.png)
无壳elf64位
扔进ida里面找到main函数
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110217.png)
一大长串还是比较复杂的
放进linux里面./跑一下
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110218.png)
告诉我们在尝试。
那我们只能转图去看ida了。一点点的分析吧，
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110219.png)
可以看到是一个输出函数
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110220.png)
应该是把输入的内容给了v3
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110221.png)
这是后面a2的出现内容，可以看到这里有个取地址，那很有可能是我们输入的内容给了这个v12
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110222.png)
又把v12的内容给了v11
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110223.png)
v11的头又给了v10的头
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110224.png)
这里不断把v11的尾部给v13，再和v10作对比
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110225.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110226.png)
可以看到这里在判断传的字符是不是最后一个字。
主要看里面的判断
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110227.png)
哦意思就是我们输入的东西和这个所谓的off_6020A0[dword_6020C0[v14]]是否相等，点进去具体看一下
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110228.png)
是一个相当长的东西
另外一个
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110229.png)
存了很多数字，并以3个0为填充。
基本上明白了，我们输入的东西应该是第一个字符串当中以第二个中的数，拼起来的一个字符串，写脚本
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110230.png)
得到flag
#题目secret-galaxy-300
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110231.png)
##解法
这道题也是一个很抽象的题目，给了我们三个文件，一个elf64，一个elf32，一个win32.
win32的我电脑跑不起来，拿到kali里面跑一下elf格式的。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110232.png)
给我了个这么东西，很没头绪。
看了别人的wp，恍然大悟，题目隐藏的信息，那应该还有一个星系啊！打开ida追踪到星系名
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110233.png)
这货没出现过，看一下在那些函数用过他。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110234.png)
是这么一个函数，这个应该就是我们要找的flag了，使用动态调试
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110235.png)
跟进按a显示字符串，出现flag。
也可以用odg
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110236.png)
#题目elrond32
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110237.png)
##解法
exeinfo打开看一下，![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110238.png)
用kali运行一下
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110239.png)
大概知道会有什么回显信息。用ida打开看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110240.png)
过程很了然，一个判断函数是关键。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110241.png)
输入的a2是0，也就是到了![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110242.png)
这个地方，如果我们输入的是i也就是105的话，就会跳转到
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110243.png)
a2变成7，到了115，也就是u，以此类推，要输入的应该是isengard。
我们直接输入
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110244.png)
flag出现。
也有别的办法。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110245.png)
flag是通过这个地方的运算才出来的是什么东西。v2为何![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110246.png)
一个数组。a1也就是我们输入的东西isengard
写个脚本flag也能出现![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110247.png)
#题目hackme
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110248.png)
##解法
又给我看超载了，直接去看了别人的wp。
先是kali中运行。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110249.png)
ok有回显，ida通过字符串锁定函数位置。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110250.png)
东西很多慢慢来，从下往上。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110251.png)
要让v21为1，就要让v16等于v19^15
再往上看，v16是什么![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110252.png)
跟进看看![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110253.png)
一个数组
那它的下标是什么？![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110254.png)
看别人的wp，跟进去慢慢看发现这是一个随机数！
很迷茫，再看看v15![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110255.png)
涉及到v11![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110256.png)
跟输入有关，应该是我们输入的东西存储的地方。下标居然也是v17？
好，这题出的很飘逸啊。他是将我们输入的东西，去随机数k为下标，以同样的k去文件自己的字符串同样的位置是不是一样的，原来如此。那我们就不用去纠结v17的取值就行了。唯一要注意的也就是为什么是22以内的，看到文件自带的字符串反应过来原来我们要输入的就是22位的！
下面就是一些处理过程了，因为跟主逻辑没什么关系，在进行异或的时候直接照抄就可以了。
写出脚本
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110257.png)
得到了flag
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020110258.png)
