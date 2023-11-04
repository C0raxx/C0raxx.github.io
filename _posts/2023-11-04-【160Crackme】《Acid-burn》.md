---
layout:     post
title:      【160Crackme】《Acid burn》
subtitle:   Crackme
date:       2023-11-04
author:     Corax
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 【Crackme】
    - 【逆向】
---

# Acid burn

这个有两个需要crack掉的，一个是左边的name和serial，另外一个是右边的serial。一点点来分析好了。

![image-20231105003633773](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311050253214.png)

## 去NAG

![image-20231105003802660](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311050253215.png)

这玩意烦的很，去掉先。

双击这里![image-20231105003916552](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311050253216.png)

来到这里，下个断吧。![image-20231105003940882](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311050253217.png)

根据栈的信息找到调用签的信息就是这个call

![image-20231105005104081](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311050253218.png)

下个断再来，可以到这个也是函数

![image-20231105005548758](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311050253219.png)

根据栈信息找到调用这里的代码

![image-20231105005615343](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311050253220.png)

可以看到这里有个je，改成jmp，直接就有了

![image-20231105005920459](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311050253221.png)

复制到可执行文件。![image-20231105010009917](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311050253222.png)

直接打开就没有NAG了。![image-20231105010103328](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311050253223.png)

# 单Serial

点进去是这样

![image-20231105011115699](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311050253225.png)

信息是Try again!!

![image-20231105011122439](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311050253226.png)

放进od 搜索字符串，有三tryagain字符串，根据两个感叹号锁定到第一个

![image-20231105011238408](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311050253227.png)

进去之后看到有一个call后面跟着一个跳，断点下上面一些

![image-20231105022319217](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311050253228.png)

运行一下可以看到这里local4和local3分别是堆栈里面的两个字符串，123456是我们输入的，另外一个是程序传入比较函数的，我们认为他就是serial。试试"Hello Dude!"

成功

![image-20231105022751581](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311050253229.png)

## serial和name

同样是字符串定位，

![image-20231105022850472](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311050253230.png)

一样，call加关键跳![image-20231105022913090](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311050253231.png)

下个断 让他跑跑

![image-20231105023021715](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311050253232.png)

已经可以看到不少关键信息了，比如右下角的那个cw-...字符串，没事，先记着我们先爆破一下。成功

![image-20231105023112504](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311050253233.png)

再用用刚刚那个字符串

0019FB00   0236A5D4  ASCII "CW-8118-CRACKED"

![image-20231105023342318](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311050253234.png)

成了，但这题还稍微多一步，它是根据name生成serial，我们换个name就不一样了。那么接着往上面琢磨。在这里下个断

![image-20231105023435195](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311050253235.png)

这里有个跳，对前四位name字符做好处理后判断了一下，如果不足四个字符就直接tryagain了

![image-20231105024048460](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311050253236.png)

这里是关键，代码取第一个字符，然后和431750被赋值的0x29相乘，然后add自己，作为数字

![image-20231105024616241](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311050253237.png)

可以写keygen了。

```
sName="axxxx"
nTemp=ord(sName[0])*0x29
sKey=str(2*nTemp)
sAns="CW-"+sKey+"-CRACKED"
print(sAns)
```

![image-20231105025236841](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311050253238.png)

成功

![image-20231105025312056](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311050253239.png)