---
layout:     post
title:      【攻防世界逆向】《Hello, CTF》《re1》《simple-unpack》《logmein》
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

#题目Hello, CTF
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114894.png)
##解法
题目这里写了，flag不一定是明文显示的，所以这个题目当中的一些中长的十六进制或十进制字符串就比较关键，用IDApro打开看一下。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114895.png)
也就是这里的437261636b4d654a757374466f7246756e
找一个网站转换一下就可以得到
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114896.png)
得到了flag
#题目re1
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114897.png)
##解法一
这是我用的方法，先用exeinfo打开看一下![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114898.png)
是比较熟悉的类型，先用ida打开看看strings，发现没有。
我以为是以前做的那种，输入一个字符串出来正确的flag，并且main函数很简单，所以想用动态调试看看，故打开odg。根据ida当中显示的位置，在判断处下了断点，更改jnz为jz（je），饶是绕过去了，但没有给我flag，所以失败了。
然后我看了一下f5给的伪代码。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114899.png)
可以看到这个v5很关键，根据位置找到这个![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114900.png)
这个两个地方很关键，应该存着flag。所以在第二个地方下断点，看看消息栏。得到flag
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114901.png)
成功了
##解法二
看wp，十分简单，因为这个题目实际上函数没有对这个码进行处理，找到码的位置，按A，就得到
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114902.png)
#题目
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114903.png)
是一个加了壳的题目，完全不会，直接看wp了。
##解法
跟教程把upx脱壳给下载下来了https://blog.csdn.net/qq_53532337/article/details/120571437
然后就是根据教程upx -d指令把壳给脱了
再看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114904.png)
已经没有壳了。再丢进ida里面，看main函数
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114905.png)
得到flag
#题目logmein
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114906.png)
这道题涉及一些算法。
##解法
这道题我解的并不轻松，用exe看一下
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114908.png)
是一个elf文件，不好运行直接丢到ida64位看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114909.png)
并f5看一下伪代码
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114910.png)
可以看到这道题的算法并不是很复杂，并且关键的字符串":\"AL_RT^L*.?+6/46"已经给出。按道理来讲只要编写脚本即可。
但我这里的问题出在对编程掌握的不够深入，因为个人没学C，学的是C++等别的语言。所以对于这个伪代码例如byte等地方不理解。我就用自己的方式写脚本。
然后又是编程的生疏，我觉得ida这里的反汇编出错了，因为在实际的C++操作中\这个符号不好表示，我就想去看看汇编代码。然后不知道为什么还真看到这里前面的字符没有表示出来，就天真的认为是ida反汇编的不可靠。于是乎AL_RT^L*.?+6/46这样来进行运算。答案当然是错了。
累计经验：反汇编需要更多经验，编程不能放掉。
最后奉上别人的wp![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114911.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114912.png)
