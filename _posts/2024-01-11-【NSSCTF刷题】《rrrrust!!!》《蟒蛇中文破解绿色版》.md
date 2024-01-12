---
layout:     post
title:      【NSSCTF刷题】《rrrrust!!!》《蟒蛇中文破解绿色版》
subtitle:   CTF
date:       2024-1-11
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【CTF】
    - 【逆向】
    - 【NSSCTF】
---

# 题目

![image-20240112134721047](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401121432985.png)

## 解法

一道rust语言逆向题

先exeinfo看看

![image-20240112134833244](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401121432986.png)

无壳elf 进ida64

进main函数

![image-20240112135109663](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401121432987.png)

进去之后可以直接看到一堆数据，感觉像密文

![image-20240112135210429](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401121432988.png)

汇编

![image-20240112135222319](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401121432989.png)

再往下看到一个比较，

![image-20240112135703559](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401121432990.png)

进汇编

![image-20240112135733522](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401121432991.png)

不确定是不是关键处，不妨动调一下

（注意一下这个 jnz要改成jz，这样才可以一轮轮循环下去）

可以看到箭头指的地方密文被传进来了，而上面穿的是什么呢？

![image-20240112140839788](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401121432992.png)

只能是加密后的明文了

![image-20240112140909871](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401121432993.png)

那现在就是要找到怎么加密的明文了，主函数很难看，所以根据这里的动调看怎么存进去的

所以动调往上找一下

发现这里的rdx有数据了，往上找

![image-20240112141335394](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401121432994.png)

来到这

![image-20240112141415186](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401121432996.png)

直接f5反编译 到这里 可以看到一个异或 密文是XFFTnT

![image-20240112141433469](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401121432997.png)

所以可以写脚本了

```
enc = [0x3E, 0x2A, 0x27, 0x33, 0x15, 0x03, 0x3D, 0x77, 0x25, 0x64,
  0x03, 0x67, 0x07, 0x32, 0x76, 0x0B, 0x1C, 0x21, 0x2B, 0x32,
  0x19, 0x23, 0x5E, 0x26, 0x69, 0x22, 0x3B]
key = 'XFFTnT'
flag = []
for i in range(len(enc)):
    flag.append(chr(enc[i] ^ ord(key[i%6])))
print(''.join(flag))
```

# 题目

![image-20240112142139787](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401121432998.png)

## 解法

下出来就是一堆py （ans我自己写的）

![image-20240112142546527](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401121432999.png)

挺简单的 主要其实就是main

过程就是取flag逐个字符，偶数位除以2  奇数位除以3 然后出下面列表

![image-20240112142723703](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401121432000.png)

就是下面的列表就是密文 所以脚本思路就是 偶数位×2 奇数位×3.

```
sEnc = ['39/1', '83/3', '83/2', '67/3', '42/1', '70/3', '123/2', '116/3', '52/1', '49/3', '115/2', '95/3', '49/2', '115/3', '95/2', '112/3', '121/2', '95/3', '119/2', '37/1', '57/1', '36/1', '50/1', '11/1', '95/2', '35/1', '58/1', '35/1', '115/2', '39/1', '26/1', '34/1', '97/2', '36/1', '33/2', '125/3']
nTemp1 = []
nTemp2 = []

for i in sEnc:
    sTemp1 = i.split('/')[0]
    sTemp2 = i.split('/')[1]
    nTemp1.append(eval(sTemp1))
    nTemp2.append(eval(sTemp2))

print(nTemp1)

'''
for i in range(len(nTemp2)):
    if i % 2 == 0:
        if nTemp2[i] == 1:
            nTemp2[i] = 2
    if i % 2 == 1:
        if nTemp2[i] == 1:
            nTemp2[i] = 3 

print(nTemp2)
'''
print(nTemp2)
for i in range(len(nTemp1)):
    if i % 2 == 0:
        if nTemp2[i] == 1:
            print(chr(nTemp1[i] * 2), end='')
        else:
            print(chr(nTemp1[i]), end='')
    if i % 2 == 1:
        if nTemp2[i] == 1:
            print(chr(nTemp1[i] * 3), end='')
        else:
            print(chr(nTemp1[i]), end='')




'''
string = '39/1'
result = string.split('/')[0]
print(result)
'''

```

出`NSSCTF{th1s_1s_py_world!_itisu4fal!}`