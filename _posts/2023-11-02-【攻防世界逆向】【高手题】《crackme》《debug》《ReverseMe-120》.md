---
layout:     post
title:      【攻防世界逆向】【高手题】《crackme》《debug》《ReverseMe-120》
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

#题目crackme
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020112907.png)
##解法
第一次做这样的题，可以说是奇难了。
`1.手动脱壳，这道题是nspack，没见过这个壳，去找别的师傅学习了一下，https://blog.csdn.net/xiao__1bai/article/details/120230397讲的非常详细了，但我还是想用我的语言把我的理解再复述一遍。nspack，北斗壳，也是一种压缩壳加密壳。用的是堆栈平衡原理，这个原理就是要在加壳的时候保证壳初始化的现场环境，就会有pushad，pushfd这样的指令。可以把壳当做一个子程序，当壳把代码解压前后，要遵循这个堆栈平衡的原理。`
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020112909.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020112910.png)
下面开始详细步骤，打开odg。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020112911.png)
ok很标准
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020112912.png)
对esp右键数据窗口追踪
按f7两步步入这时候两个push已经压栈完毕，可以看到esp现在所指向的地方![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020112913.png)
对这个地方下硬件断点
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020112914.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020112915.png)
执行过来是这样的界面，可以看到下面一行就要call了。我们步入。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020112916.png)
来到了这里就是我们的oep。用dump工具进行脱壳
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020112917.png)
再用ida打开看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020112918.png)
已经是脱完壳的状态了。
算法也很简单，但不知道为什么，这里的print和scanf没给我识别出来，但也无伤大雅吧。直接写出脚本
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020112919.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020112920.png)
#题目debug
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020112921.png)
##解法
这道题也是从来没做过ida反汇编出来是这个东西
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020112922.png)
一时间也盲头了。直接去看别人的wp。
`2.是要用.net反编译去做，有两款软件ILSPY和dnSPY，我用的是dnSPY。在dnspy中几个关键操作是：F9为添加新的断点；F5为调试程序集；F10为逐过程运行程序语句；F11为逐语句调试；shift+F11为跳出。`
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020112923.png)
打开之后找到关键函数的地方。然后分析一下。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020112924.png)
这里主函数已经读入我们的内容之后与他内置的flag进行一个对比，所以我们直接就在比较的地方下断，看一下存的到底是什么flag
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020112925.png)
直接看到flag
#题目ReverseMe-120
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020112926.png)
##解法
这题没有前面两题这么抽象，ida分析就可以看到过程
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020112927.png)
判断正误的就在于红框内的代码。那我们就要看v12在哪里动了手脚了。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020112928.png)
很奇怪，居然没有传入。确实是很蹊跷点击进入这个才恍然大悟。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020112929.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020112930.png)
`3.这里就有一个知识点，在外面的时候可以看到只是对v11进行了操作，但是点进去之后才发现这是对寄存器进行传参了，只是ida识别不出来而已。`
ok，那么这个函数就是比较关键的处理函数。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020112931.png)
这里我没看懂，看了别人的writeup，才明白这里是一个base64加解密的过程，关键可以看看这里
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020112932.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020112933.png)
所以逆向思路也清晰了，you_know_how_to_remove_junk_code，异或操作，base64加解密
贴出脚本
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020112934.png)
总结：
crakeme 脱壳题 栈堆平衡
debug 初识.net
ReverseMe-120函数传参 base64
