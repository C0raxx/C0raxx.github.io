---
layout:     post
title:      【160Crackme】《hacktooth crackme #3》（未完全解出）
subtitle:   Crackme
date:       2023-11-28
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【Crackme】
    - 【逆向】
---

《hacktooth crackme #3》

今天这道CM，我看了一天，诸多不顺。

到最后也没解出来。想着把这个CM的步骤给写一下吧，后面等别人的出了之后参考参考。

就昨天下午的一个新的CM。

![image-20231129211640476](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200842.png)



下载下来是一个exe

![image-20231129211713062](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200844.png)

打开看看

一个比较传统的样式。

![image-20231129211730240](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200845.png)

拿exeinfo看一下信息

无壳的，IDA看看



![image-20231129211807408](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200846.png)



进来直接就是main函数

如图所示

![image-20231129211915665](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200847.png)

流程比较复杂，想从string来入手

![image-20231129212125748](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200848.png)

定位到registered

也就是这个地方看看流程

简单来说就是之前做了一个验证的过程

然后改变标志位，然后根据cmovz决定是否把ecx的内容覆盖进去。

扔OD动调一下

![image-20231129212207203](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200849.png)

直接nop掉

![image-20231129212816000](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200850.png)

爆破成功

![image-20231129213007615](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200851.png)



分析算法

首先要做两件事情

第一个 原本放进OD不是401000开始的，因为这在vs2010后面写的。

到这里改一下这个值

把这玩意的值改成0

就可以了

![image-20231129213211845](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200852.png)

另外一个，本题题目说有一个简单的反调试，我看string里面有一个

![image-20231129213400544](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200853.png)

跟进

![image-20231129213453573](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200854.png)

就在这个函数里面调用

![image-20231129213512138](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200855.png)

他是在初始化的函数里面的

想办法patch掉。

注意到就给了v8

后面有个用到v8的地方

![image-20231129213547163](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200856.png)

根据v8

403b06

![image-20231129214035864](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200857.png)

![image-20231129214052286](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200858.png)

patch成75就可以了

![image-20231129214110648](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200859.png)

然后另存为这个就可以了。



其实最开始后看的时候没注意到isdebugger，也不确定是否只有这个反调试

最开始我以为是用线程的方式开的

网上找了很多，看着都不像，就往调用链上面看。



![image-20231129214807051](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200860.png)

这里面调了main

又往前面看了一些，由以为是互斥锁搞得鬼。

![image-20231129215036467](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200861.png)

不由得只能动调



可以看到我下的断的过程。

![image-20231129215047244](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200862.png)

最开始从这里

这里跳到输出字符串



![image-20231129215133230](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200863.png)

发现已经备好了

![image-20231129215201992](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200864.png)

字符串的位置在

4080e0

![image-20231129215218533](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200865.png)

我就想在这下个内存写入断点

注意这里要shift f9启动，忽略一些错误

来到这里

这条是关键

![image-20231129215335488](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200866.png)

跟随一下地址

![image-20231129215343236](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200867.png)

发现这个地址b5fe90

重启下断点

告知无效

![image-20231129215509105](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200868.png)

看一下m

![image-20231129215535204](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200869.png)

开始的时候还没有申请这块区域，里面有个用malloc开过，ida看看

![image-20231129215627126](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200870.png)

![image-20231129215634537](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200871.png)

确实如此

而且调用了很多遍

根据ida信息找到

这块区域

![image-20231129215832062](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311292200872.png)

在里面下断点。

然后后面的步骤更复杂，我看了看，应该是用虚函数 对密文处理，存放在一个另外开辟出来的7b开头的区域，然后逐个读入b5fe90。然后再进入004080E0的。

但是调用链太复杂，可能我掉了哪个坑，等后续别人的解。