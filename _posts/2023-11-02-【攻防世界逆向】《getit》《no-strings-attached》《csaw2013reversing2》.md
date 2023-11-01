---
layout:     post
title:      【攻防世界逆向】《getit》《no-strings-attached》《csaw2013reversing2》
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

#题目getit
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114484.png)
##解法
先用exeinfo打开看看文件格式
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114485.png)
无壳elf文件，放进ida64打开看看，并f5查看伪代码
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114487.png)
耐心的学习了一下。我先学了下面的文件操作
fseek 修改原指向stream流指针，按照第p【i】个位置 从左开始数
fputc 把前面内容 从上面的指针开始编辑 不带格式化
fprintf 把内容写入流文件 带格式化
到这里文件的操作已经分析完了，转手看把什么字符串进行了处理。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114488.png)
看的时候遇到了一些困难，比方
lodword有啥用：其实和正常的调用没啥区别
v5&1 可以说是判别正负号
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114489.png)
这里的部分是对t进行操作，这样子我们可以写脚本了。
b70c59275fcfa8aebf2d5911223c6589 加上外面的SharifCTF{}即可
#题目no-strings-attached
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114490.png)
##解法
这道题也是一道elf题目，有一个方法是用gdb来动态调试，但我的电脑还没有装好gdb，所以只能用ida来逆向。
进入之后![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114491.png)
可以看到这样的几个函数，一个个打开，这个比较可以
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114492.png)
锁定里面的decrypt
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114493.png)
可以看到里面的一下加密过程，所以只要知道传进来的什么就可以了。往前，点击
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114494.png)
跟进可以看到字符串，编写脚本即可
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114495.png)
#题目csaw2013reversing2
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114496.png)
##解法
这题真是烧了我太多脑细胞（终究是太菜了）。
打开它![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114497.png)
乱码
无壳32位程序，直接放进ida进行伪代码
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114498.png)
再一个个打开，debugbreak（）是debug，重点在于这个sub40100打开看一看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114499.png)
好，是一个加密过程，过程有点长，试试看动态调试
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114500.png)
重点在于这个地方（我已经汇编完了）。红字的地方雨哪里是jz，意思就是后面的B9（也就是40100）根本没执行，把它改一改。还是不行。
问题出在这
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114501.png)
这是40100中的部分，按照他原来的跳转地址输出的flag地址是错误的（因为![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114502.png)
）这里不一样，改成图中的地址，运行。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020114503.png)
出现了flag。