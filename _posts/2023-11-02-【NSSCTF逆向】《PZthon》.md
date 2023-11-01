---
layout:     post
title:      【NSSCTF逆向】《PZthon》
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

题目到手是一道EXE
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020143858.png)
打开来是这个样子，输入flag判对错
查壳
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020143859.png)
没壳，直接扔ida里面
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020143860.png)
。
。
是一些python的信息。
开始还没想到解包，想就着这个程序流程分析下去了。
。
走正确的路吧
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020143861.png)
用工具解包exe
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020143862.png)
有了这个pyc之后，然后
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020143863.png)
https://tool.lu/pyc/
就这个网站

```
def hello():
    art = '\n              ___                                                                      \n    //   ) )     / /    //   ) )  // | |     / /        // | |  \\ / / \\    / /       \n   //___/ /     / /    //        //__| |    / /        //__| |   \\  /   \\  / /        \n  / ____ /     / /    //  ____  / ___  |   / /        / ___  |   / /     \\/ /         \n //           / /    //    / / //    | |  / /        //    | |  / /\\     / /          \n//           / /___ ((____/ / //     | | / /____/ / //     | | / /  \\   / /           \n                                                                                       \n     / /        //   / / ||   / / //   / /  / /       /__  ___/ ||   / |  / / //   ) ) \n    / /        //____    ||  / / //____    / /          / /     ||  /  | / / //   / /  \n   / /        / ____     || / / / ____    / /          / /      || / /||/ / //   / /   \n  / /        //          ||/ / //        / /          / /       ||/ / |  / //   / /    \n / /____/ / //____/ /    |  / //____/ / / /____/ /   / /        |  /  | / ((___/ /     \n'
    print(art)
    return bytearray(input('Please give me the flag: ').encode())

enc = [
    115,
    121,
    116,
    114,
    110,
    76,
    37,
    96,
    88,
    116,
    113,
    112,
    36,
    97,
    65,
    125,
    103,
    37,
    96,
    114,
    125,
    65,
    39,
    112,
    70,
    112,
    118,
    37,
    123,
    113,
    69,
    79,
    82,
    84,
    89,
    84,
    77,
    76,
    36,
    112,
    99,
    112,
    36,
    65,
    39,
    116,
    97,
    36,
    102,
    86,
    37,
    37,
    36,
    104]
data = hello()
for i in range(len(data)):
    data[i] = data[i] ^ 21
if bytearray(enc) == data:
    print('WOW!!')
else:
    print('I believe you can do it!')
input('To be continue...')

```
程序流程很明显
就是我们输入的字符串异或21，然后和enc比较。
逆向思路就是
```


enc = [
    115,
    121,
    116,
    114,
    110,
    76,
    37,
    96,
    88,
    116,
    113,
    112,
    36,
    97,
    65,
    125,
    103,
    37,
    96,
    114,
    125,
    65,
    39,
    112,
    70,
    112,
    118,
    37,
    123,
    113,
    69,
    79,
    82,
    84,
    89,
    84,
    77,
    76,
    36,
    112,
    99,
    112,
    36,
    65,
    39,
    116,
    97,
    36,
    102,
    86,
    37,
    37,
    36,
    104]
for i in enc:
    print(chr(i^21),end='')

```
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020143864.png)
