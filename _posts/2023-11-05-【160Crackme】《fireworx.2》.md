---
layout:     post
title:      【160Crackme】《fireworx.2》
subtitle:   Crackme
date:       2023-11-05
author:     Corax
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 【Crackme】
    - 【逆向】
---

# fireworx.2

exeinfo查一下壳，无壳 用borland写的

![image-20231106003418104](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311060118110.png)

打开来看看，字符字样就这样

![image-20231106003455046](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311060118111.png)

扔od吧

查字符串，可以看到是明文，直接找到关键字符串。

![image-20231106003539394](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311060118112.png)

定位到关键处之后，可以看到这关键跳。

![image-20231106005624423](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311060118113.png)

尝试通过nop来爆破他。成功了。

![image-20231106005758466](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311060118114.png)

逆一下算法吧。在上面这个地方下个断，因为有很多push，pop做栈堆平衡

![image-20231106010657448](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311060118115.png)

两个红框内的算法负责把输入的name和serial给读出来。

运行到这里。

把关键内容给压栈

![image-20231106010955418](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311060118116.png)

然后调用一个程序，拼接起来

![image-20231106011051716](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311060118117.png)

流程就是复制两遍，加上625g72



写注册机

```python
sName="axxxx"

sKey=sName+sName+"625g72"

print(sKey)
```

运行结果

![image-20231106011616098](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311060118118.png)