---
layout:     post
title:      【NSSCTF刷题】《moveAside》
subtitle:   CTF
date:       2024-1-5
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【CTF】
    - 【逆向】
    - 【NSSCTF】
---

哎呀，这段时间把学校里的一些考试之类的事情解决了，后面实习的事情也暂时谈好了。这个寒假就狠狠冲一波吧，后续更新的内容会是一些关于CTF，病毒分析，内核啊，部分安卓的内容，还会写一些自己的思考，如何与自己相处和与世界相处的思考。



好了进入正题

# moveAside

![image-20240106165152876](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401061837833.png)

## 解法

这是一道2023CISCN的国赛题目，半年前就没做出来，现在还是没啥思路。近期故痛定思痛，看了很多师傅的做法，对这个LD_PRELOAD的方法比较感兴趣，手动复现了一下。

首先看看文件信息

![image-20240106165344906](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401061837834.png)

32位ELF，放进Ubuntu看看

效果是，我们输入第一个123456，然后如果不正确的话会让我再输入两个东西，不报错的结束，正确的话会输出yes。

![image-20240106165432689](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401061837835.png)

既然没壳，直接IDA

看到第一眼感觉左边函数很少呀！但右边扫了一眼，全是mov，使用了mov混淆，有点恶心了。

![image-20240106165556663](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401061837836.png)

没啥思路，看看字符串

![image-20240106165701012](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401061837837.png)

点进去，可以看到下面有个yes，点进去，一个样子。

![image-20240106165724129](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401061837838.png)

![image-20240106165727196](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401061837839.png)

然后这位师傅的方法是这样想的，因为strcmp已经被分析出来了，仅仅是一个比较的功能

![image-20240106170520952](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401061837840.png)

通过LD_PRELOAD设置环境变量的方法，可以先行加载一个我们自定义的Strcmp，比如下面这个

```
#include<stdio.h>

int strcmp(const char* s1, const char* s2)
{
	printf("strcmp called\n");

	while (*s1 && (*s1 == *s2))
	{
		++s1;
		++s2;
	}

	return *s1 - *s2;
}
```

编译成.so后，加载进去，然后打开程序

就是如下命令

![image-20240106170741332](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401061837841.png)

我们可以看到当我们输入完全不一样的flag之后就只有一次strcmp called，而前几位正常的话，可以看到会由多次，而正确flag的个数相当于strcmp called次数减1。

![image-20240106170839508](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401061837842.png)

然后我们有了一个思路，我们先命名一个字符串空`ans = '' `,接着假设第一轮我们输入1，然后错误回显信息就是一条strcmp called，这个回显信息的条数显然小于我们输入字数+2即`len(ans)+2`

举个例子，我们现在输入f

回显如下，三条，而目前我们的ans还没有加入第一个正确字符s，所以

![image-20240106174532099](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401061837843.png)

回显3条大于`len(ans)+2`也就是大于0+2，要注意ans现在还没有加入f！！



那么思路就出来了

1. 把自定义的strcmp先载入
2. 新建进程，根据回显调数判断是否此轮字符是否正确
3. 不断新建进程



自定义的strcmp已经写好编译完成，python脚本如下

```
from pwn import *
import string

context.log_level = 'error'

ans = ''

for i in range(42):
    for ch in string.printable:
        current_flag = ans + ch
        print(current_flag)
        p = process('./moveAside')
        p.recvline()
        p.sendline(current_flag.encode())
        recv = p.recvall(timeout = 0.01)
        recvs = recv.splitlines()

        if len(recvs) > len(ans) + 2:
            ans += ch
            break
```

跑一下

![image-20240106174953470](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401061837844.png)

出flag，也蛮快的。

不知道这是不是算测信道攻击的一种呢？还看了pintools加python爆破的思路

还有直接在原strcmp直接换表的，给我看懵了。

https://www.cnblogs.com/gaoyucan/p/17440735.html

后面会写个关于LD_PRELOAD和pintools的博客。

