---
layout:     post
title:      【160Crackme】《CrackMe3》《cosh.3》
subtitle:   Crackme
date:       2023-11-19
author:     Corax
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 【Crackme】
    - 【逆向】
---

# CrackMe

没有难度的CM，没什么意思

exeinfo

无壳32位

![image-20231120105915071](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311201146100.png)

扔ida看看

目标是正确字符串，那么前面有两个jnz跟着判断

![image-20231120110003111](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311201146101.png)

## 爆破

那就在两个jnz地方汇编成jz就可以了。

![image-20231120110301495](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311201146102.png)

爆破就成功了

![image-20231120110327000](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311201146103.png)

## 算法逆向

其实没有算法

根据ida里面的信息，在第一个比较的地方下断点

![image-20231120110420068](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311201146104.png)

再在第二个地方下断点

![image-20231120110456327](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311201146105.png)

输入如下信息

![image-20231120110606003](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311201146106.png)

呃呃，看寄存器里面传的参，一个是输入的字符串，另外一个是

00441014=CrackMe3.00441014 (ASCII "**Registered User**")

所以name就是这个？

改一下zf，接着往下走

然后把754开头的serial和GFX开头的serial作比较，这俩只是调换次序而已。

![image-20231120110758369](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311201146107.png)

打开程序看看

![image-20231120110838624](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311201146108.png)

成功。。。



# cosh.3

这个算法部分稍微复杂点。

exeinfo 无壳32进ida

![image-20231120113621567](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311201146109.png)

## 算法逆向

感觉流程稍微有点复杂

![image-20231120113704646](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311201146110.png)

![image-20231120113715071](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311201146111.png)

刚开始做的时候比较大意，在这里就下了断点

![image-20231120113758622](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311201146112.png)

后面输入如下

![image-20231120113814067](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311201146114.png)

根本到不了断点处，估计前面上面有东西断下来了。往上看

![image-20231120113847845](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311201146115.png)

和5比较，再jg，前面还有获取文本长度的函数，很明显，这个就是判断长度然后分析跳转

那我们输入六个字符，再进OD看看

![image-20231120114029061](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311201146116.png)

直接进入第一个处理

![image-20231120114040385](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311201146117.png)

流程很简单，就是读入每一个字符，第一个是h，然后和1做异或，到了第二个y，就是和2做异或

![image-20231120114133143](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311201146118.png)

到第二个循环，也是一样的，只不过换成了第一轮和10异或，然后11…

![image-20231120114216817](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311201146119.png)

然后后面进入比较部分

循环比较，不对就跳到错误回显

![image-20231120114233651](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311201146120.png)



由此可以写keygen了，思路就是对已有字符串进行0x1到0x字符长度+1的异或，得到的字符串再异或0xa到0x字符长度+a,也就是异或出来，就是我们的serial。

如下

```
sName = "hyyyyy"
sEncName = ""
sSerial = ""

for i in range(1,len(sName)+1):
    sEncName += chr(ord(sName[i-1]) ^ i)

for i in range(10,len(sName)+10):
    sSerial += chr(ord(sEncName[i-10]) ^ i)

print(sSerial)

```

跑跑看

![image-20231120114432362](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311201146121.png)

成功

![image-20231120114458532](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311201146122.png)



## 爆破 

忘记爆破了，我的思路就是直接在错误会显得地方，跳到正确的地方

![image-20231120114622651](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311201146123.png)

成功