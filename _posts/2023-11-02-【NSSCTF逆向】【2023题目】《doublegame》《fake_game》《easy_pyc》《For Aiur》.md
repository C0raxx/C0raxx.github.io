---
layout:     post
title:      【NSSCTF逆向】《doublegame》《fake_game》《easy_pyc》《For Aiur》
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

#总览
doublegame 迷宫 游戏程序逆向
fake_game exe解包 pyc反编译
easy_pyc pyc反编译 rsa解密
For aiur exe解包 pyc反编译 Anaconda pycdc
#题目doublegame
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137786.png)
##解法
感觉还是蛮抽象的一题
打开看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137787.png)
是一个贪吃蛇，也不懂啥直接放进ida看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137788.png)
有很多函数，不想一个个看了，直接看string
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137789.png)
感觉有很多有用的信息，题目信息又说是doublegame所以应该还有一个游戏，看红框的内容应该是这个迷宫了，点进去通过交叉引用看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137790.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137791.png)
ok就是一个迷宫，上面的图也说了flag跟路径有关，我们就解解看
dddssssddwwwwddssddwwwwwwddddssa
就是这一串，取md5值然后加上score就可以了，score是什么呢？
这个迷宫应该是没啥分数讲的，要么就是贪吃蛇的分数。想着可以通过伪代码看看怎么调用第二个游戏的。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137792.png)
不同地查看交叉引用，最后发现在这个地方是判断是否调用第二个游戏的。那这个score很有可能是这个c
于是NSSflag{0d9eb6b95c042f6649fb9d4ab062972e13371337}很可惜不对，不解到底是path有问题还是这个score有问题呢？
查看了别人的flag，原来在调用第二个游戏的时候达到*后还有这样的提示
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137793.png)
还要带出去，所以我的path是有问题的
所以正确的flag就是NSSCTF{811173b05afff098b4e0757962127eac13371337}
。
。
其实这道题估计附件下载下来可能是有啥问题
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137794.png)
这个地方其实还缺了0，也就是墙。
#题目fake_game
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137795.png)
##解法
这道题是给了一个exe游戏，植物大战僵尸，玩了十多分钟没给flag做题
反编译它出来一坨屎，看都看不懂。无奈只能看别人的wp，但也学到了很多新知识。
`这个exe文件是py文件打包的，所以我们要做的就是先通过一个软件叫做pyinstxtractor.py。然后和exe放在一个文件夹下，然后在文件目录打开`
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137796.png)
在extracted打开就有了一个pyc文件，依旧是熟悉的uncompyle6
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137797.png)
出来了一个py文件
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137798.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137799.png)
经过如图的处理就可以得到flag。下面那个是四元一次方程组，计算器算
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137800.png)
然后编写脚本
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137801.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137802.png)
奇奇怪怪，上网看一下。他们的这个最后一个是2361，而我自己验算的时候发现如果代入1则会出现小数位的5。不太明白。
flag NSSTF{G0Od_pl2y3r_f0r_Pvz!!}
#题目easy_pyc
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137803.png)
##解法
这道题没有以前做的pyc那么恶心，没有混淆。
首先还是熟悉的反编译成py。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137804.png)
然后打开
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137805.png)
很显然这是一个rsa的解密题。
找找脚本
看到一个师傅的blog我觉得在rsa这方面讲的很详细
https://blog.csdn.net/qq_45521281/article/details/114706622
然后编写脚本
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137806.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137807.png)
出flag
#题目For Aiur
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137808.png)
##解法
这题也是我litctf上的题目，但是我也没做出来，看了别的师傅的wp找了些灵感
这道题跟上面那题是同种类型，都是exe解包pyc反编译py的逆向
所以同样还是上pyinstxtractor.py，但是这题解包还是反编译的时候出现了问题了，可能是出题人使得坏，这道题得要用py38版本进行解包，要不然会缺少关键文件。
在解决的道路上又收获了新软件。其中一个就是Anaconda，在这个程序当中可以切换不同的py环境，免去疯狂切换path的尴尬。用法在这
https://blog.csdn.net/tqlisno1/article/details/108908775?spm=1001.2014.3001.5506
而后成功进行了到pyc的解包，接下里就是反编译了。
但是不知道为什么我的uncompyle好像犯病了，tmd一直报错，于是我去看了一下别人的方案，这里又有另外一个新软件pycdc，也是一个pyc到py的反编译软件。用法在地址
https://blog.csdn.net/qq_63585949/article/details/127080253
注意在使用的时候最好使用powshell管理员模式，要不然也会有莫名其妙的报错。
于是乎，成功地反编译了![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137809.png)
打开看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137810.png)

也是一个很长的代码，但是通篇不见flag，但在文件头获取了线索，这里引用了ch文件，这个文件在![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137811.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137813.png)
同样的反编译可以看到
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137814.png)
这里的逻辑就很简单了，就是找一个num满足判断条件，转成str之后进行运算。于是编写脚本
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137815.png)

获得flag
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020137816.png)
