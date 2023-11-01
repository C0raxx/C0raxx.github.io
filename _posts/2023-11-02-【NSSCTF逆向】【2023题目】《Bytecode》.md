---
layout:     post
title:      【NSSCTF逆向】【2023题目】《Bytecode》
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

#题目 Bytecode
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020139623.png)
##解法
是蛮久没见的字节码题目，还有点记忆
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020139624.png)
这里一大长串应该是一串数组，跟存储后变形的flag有关
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020139625.png)
上面的啥打印错误什么的应该是在跑的时候告诉我们有问题，也不用管
红框里面可以看到是一个循环 从46到152
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020139626.png)
可以继续看到红框里面的东西，应该就是主要的变形过程了。
写wp
`m=[25,108,108,176,18,108,110,177,64,29,134,29,187,103,32,139,144,179,134,177,32,24,144,25,111,14,111,14]
for n in range(28):
    for i in range(46,152):
        if (i * 39)%196 == m[n]:
            print(chr(i),end='')
            break`
G00dj0&_H3r3-I$Y@Ur_$L@G!~!~
