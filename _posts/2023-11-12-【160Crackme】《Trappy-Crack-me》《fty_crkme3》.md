---
layout:     post
title:      【160Crackme】《Trappy Crack me》《fty_crkme3》
subtitle:   Crackme
date:       2023-11-12
author:     Corax
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 【Crackme】
    - 【逆向】
---





第二个好玩

## Trappy Crack me

同样是国外网站[Crackmes](https://crackmes.one/crackme/6547b4d50f4238b24302b588)最新出的一个CM。

介绍如下

![image-20231113142141020](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939650.png)

虽然说是难度2.3，但其实设计的有些问题。

### 爆破

进来一看非常简洁啊，无壳64位拿VS14写的。

![image-20231113142212293](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939651.png)

放IDA里看看吧。进来直接映入眼帘的是一串“correctcode”，试了一下，直接闪退出去了，是个假code。

![image-20231113142322246](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939652.png)

打开string看看

![image-20231113142432393](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939653.png)

跳转到如下位置，感觉这个位置可能存储了真正的key

![image-20231113142457130](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939654.png)

x64dbg跑一下，同样是有字符串用字符串定位。

![image-20231113143215988](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939656.png)

红框是字符串的位置，上面有俩跳，一个是判断Size，一个是判断字符串是否正确。那我们往上下断点。我们输入12345678，跑起来![image-20231113143257309](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939657.png)

第一个，8和5比较，显然是我们的长度，正确的应该是5.

![image-20231113143449491](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939658.png)

第二个，和+1TP3会有个比较，这是动态生成的，一个是一大长串乱七八糟代码所生成的东西。

![image-20231113143533033](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939659.png)

拿来试试

![image-20231113143644155](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939660.png)

成功了。



## fty_crkme3

这个是原160CM里面的一个，我看算法分析难度有三星，自己做出来之后还是挺有成就感的。

解压压缩包出来三个东西

![image-20231113184808721](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939661.png)

一个信息文件，一个未脱壳的exe，一个号称已脱壳的exe。

直接拿脱完壳的exe放ida里面看看

如脱，脱是脱了，但是不喜欢这个代码段字样。还是自己脱吧。

![image-20231113185321106](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939662.png)

### 手工脱

esp平衡，在esp下个段

![image-20231113185536817](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939663.png)

中转一下

![image-20231113185619514](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939664.png)

然后就来到oep了。

传统upx公式就能解决，但一些别的壳就不是了。。明天把esp平衡给再看看吧。

再dump下来

![image-20231113185821703](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939665.png)

出来一个新程序，扔进od里面，已经成功了。

![image-20231113190020058](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939666.png)

### 工具脱

upx最经典的-d大法。

![image-20231113190120903](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939667.png)

![image-20231113190130261](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939668.png)

直接拿下。

### 爆破

开始接触到程序内部了。

这道CM字符串可以直接定位

定位进去之后可以看到，错误信息的跳转源多的一比。一个个改太慢了，干脆直接在错误信息处jmp

![image-20231113190511258](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939669.png)

如下，不管输的什么，就算输错了，也会通过调用错误回显来跳转到正确回显

![image-20231113190645897](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939670.png)

跑跑看，非常ok

![image-20231113190701944](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939671.png)

## 算法逆向

还是比较麻烦的，如上所述，题目的跳转非常之多。

但是还是得看，为了方便起见，我们给错误回显处起个标签

从第一个地方444b92看起

这段代码的作用就是逐位读入所写字符串，然后和0，9的ascii比较，意图就是要输入的就是0-9之间的数字

![image-20231113191547389](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939672.png)

再接着这段，判断长短好像要9位-11位

![image-20231113191615551](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939673.png)

之后开始进入正式校验。首先是两个位置2和6，但从0开始嘛，就是第三位和第七位是不是“-”，就是说最后的key应该xx-xxx-xx的一个格式。现在我们的123456789显然不是，下个断再跑一趟。

![image-20231113191645877](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939674.png)

然后经过一大长串 内存和栈内的处理，将我们输入的12-345-67变成十六进制形式的1234567存在eax里面

![image-20231113192450440](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939675.png)

再把7给ebx

然后经过取第一个字符也就是1，经过三个函数的调用结果放进edi里面



![image-20231113192541694](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939676.png)

这个时候出来是1，目前我们对三个函数基本上一无所知，再跑

根据最后一句计算结果来看，我们需要的，关键的东西应该是存在ex里面，而eax用444b20跑出来，我们着重看一下。

应该是进行一个运算，参数是原本ebx当中的7和读到的数字，计算结果存在eax当中。

![image-20231113193028670](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939677.png)

点进函数内部看一下，就长这个样子，作用也很清楚，n次幂，这里n是7。

![image-20231113193200100](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939678.png)

出函数，发现对每一位都这么处理

![image-20231113193429223](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939679.png)

跑到最好的结果存在edi当中，我们输入的1234567是

![image-20231113193506192](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939680.png)

后面和esi也就是我们符号的数字存储作比较。

![image-20231113193517397](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939681.png)

![image-20231113193525616](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939682.png)





现在算法也算摸清楚了。

有一个7位数，每一位的7次方加在一块，等于原本的7位数。

写一个脚本（我写的这个效率比较低，大家尽情发挥）

```
nAns = [] #用来存储当前七位数的每一位
nSum = 0 #存储当前计算之和
for i in range(1000000,10000000):
    nNumber = i
    for _ in range(7):
        digit = nNumber % 10  # 获取个位数字
        nAns.insert(0, digit)  # 将数字插入数组最前面
        nNumber //= 10  # 去掉已经处理过的最后一位数字
    for m in nAns:
        nSum += m ** 7 #进行7次方
    if i == nSum:
        print(nSum)
    nSum = 0
    nAns.clear() #清空i的每一位数用作下一轮
```

跑出来四个数

![image-20231113193813928](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939683.png)

每一个都是能用的，我就演示一个，成了。

![image-20231113193840166](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311131939684.png)