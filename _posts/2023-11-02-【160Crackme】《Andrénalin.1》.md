---
layout:     post
title:      【160Crackme】《Andrénalin.1》
subtitle:   Crackme
date:       2023-09-30
author:     Corax
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 【Crackme】
    - 【逆向】
---

![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020103353.png)
这个程序上来就是要输入一个啥key
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020103354.png)
乱输一个，大概就是报错了
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020103355.png)
可以看到没有壳
扔od里面吧。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020103356.png)
可以看到一进来就是有这个msvb50的信息，是一个支持文件
直接查关键字符串吧
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020103357.png)
使用智能搜索找到几个比较看起来比较重要的字符串，看看right是咋用的。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020103358.png)
这串代码应该就是正确时候执行的，往上看有个关键跳，看看他跳到哪里去
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020103359.png)
可以看到这里应该就是跳到失败的地方了。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020103360.png)
所以这里如果采用爆破，那爆破的方法，就是把0F84改成0F85，也就是变成jne，
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020103361.png)
成功，还有一个就是直接把这句nop掉。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020103362.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020103363.png)
同样可以成功。
那如果不用爆破，完全破解咋弄呢。还可以继续往上看，看怎么决定这个跳的。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020103364.png)
可以看到执行完红框后的语句之后，我们输入的东西给到了ecx，然后压栈，再有一个字符串压栈
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020103365.png)
然后进行一个cmp，再把答案通过一个循环干涉到esi，edi
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020103366.png)
通过cmp di，si决定跳转与否。
所以key就是这个字符串
"SynTaX 2oo1"

![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020103367.png)
也成功了。
。
。
。
总结一下吧，这题很简单，甚至没有判断我们输入字符串的大小，直接nop都可以破解，不过可以拿来给学弟学妹讲讲入个门啥的还是挺不错的。
