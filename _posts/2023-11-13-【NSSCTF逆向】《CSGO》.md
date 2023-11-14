---
layout:     post
title:      【NSSCTF逆向】《CSGO》
subtitle:   CTF
date:       2023-11-13
author:     Corax
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 【CTF】
    - 【逆向】
    - 【NSSCTF】
---



# 题目

![image-20231114154439597](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311141645706.png)

## 解法

我觉得这道题还是有点难度的，涉及一些反调试，go语言逆向，还有base64换表。

用go写的程序感觉看起来都挺麻烦的。



诸如此类的函数都是go的runtime标准库写的，看起来就很麻烦。

![image-20231114155620583](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311141645707.png)



我看了其他师傅的wp，很多都是直接从函数里面直接定位到关键函数了。。我功夫还不够深，就不直接看了。



于是我们尝试使用一个识别算法的插件，在这道题是可以用的

![image-20231114155954479](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311141645708.png)



插件界面

可以看到有些关键信息了

![image-20231114160135811](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311141645709.png)



都是些关键信息

### base64下断

点进第二个

![image-20231114160228164](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311141645710.png)

![image-20231114160253865](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311141645711.png)



可以看到已经两个比较关键的函数名了，第一个enc加密，第二个比较。我们先点进去第一个

长这样

![image-20231114161440420](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311141645712.png)

我不知道为什么，可能是ida有点问题？反编译有点问题吧。查看汇编代码

这才是比较关键的函数

![image-20231114161554819](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311141645713.png)

追踪至此

![image-20231114161618843](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311141645714.png)

enc点开

有些比较关键的信息了，这个地方多轮取出了换表后的base64table，再return encode这个地方我们下个断先。

![image-20231114161634365](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311141645715.png)





## isdebugger下断

看题目描述嘛，这题有反调试，然后插件也给我们写出来了。

但是没啥信息，我们搜索函数

![image-20231114161912669](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311141645716.png)





一个个看 找到了这个

![image-20231114161945998](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311141645717.png)

一步步往上，来到了这里

![image-20231114162035035](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311141645718.png)

我的做法是不在反编译的代码处下断，因为实际上汇编代码写得是不太一样的，我们看看

汇编里面的意思其实是call完查看调试状态函数之后，结果和0比较决定jz跳不跳，不跳的话就exit了，所以我们尽量得让他跳出去，在这边也下个断点。

![image-20231114162633732](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311141645719.png)

ok现在可以动态调试了。

![image-20231114162815090](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311141645720.png)



### 动态调试

ok我们现在来到刚刚下断点进行反调试的地方了，直接修改zf为为1，让jz不进行



![image-20231114162901000](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311141645721.png)

跑，现在让我们输入一个字符串，随便输输，来到这里

![image-20231114163040990](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311141645722.png)

现在fkreverse已经配置完了，后面的fkreverse_实际上用的也是这玩意，我们直接点进去

![image-20231114163203695](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311141645723.png)

导出数据

获得了自定义的table

![image-20231114163223447](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311141645724.png)



ok现在已经拥有了自定义的table，默认的table更不用找了就是

```
ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/
```

现在要的就是密文是什么



回到最开始的main

![image-20231114163429973](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311141645725.png)

我想着既然memequal要比较，那肯定要传参吧，应该是能看看的

根据地址找到这里，没想到直接找到了，原来是硬编码写在程序里面的，最开始那张图其实也有信息，没仔细看。

![image-20231114164123329](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311141645726.png)



用一直用的脚本直接跑一跑

```
import base64

str1 = "cPQebAcRp+n+ZeP+YePEWfP7bej4YefCYd/7cuP7WfcPb/URYeMRbesObi/="

string1 = "LMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/ABCDEFGHIJK"     # 替换的表

string2 = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"

print(base64.b64decode(str1.translate(str.maketrans(string1, string2))))

# DASCTF{73913519-A0A6-5575-0F10-DDCBF50FA8CA}

```



出了





## 总结

是一道非常好玩的题，但是之前很少用ida来动调，也算在这方面更熟练了吧。