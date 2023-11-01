---
layout:     post
title:      【NSSCTF逆向】【2023题目】《enbase64》《ez_XOR》《easy_asm》
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

#题目enbase64
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135681.png)
##解法
这道题也是前几天Litctf中的一道题，也给我卡了好一会。现在我们来重新做做。
先查壳
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135682.png)
ok无壳放进ida里面看看
进去就是主函数
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135683.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135684.png)
check函数点进去
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135685.png)
很明显这里就是最后判断的地方。
是我们输入的字符串在一定的处理之后变成另外一个字符串与GQTZlSqQXZ/ghxxwhju3hbuZ4wufWjujWrhYe7Rce7ju相比较。
那我们看看我们输入的字符串做了些啥处理吧。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135686.png)
这个base64应该是比较重要的，点进去看看吧
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135687.png)
进来后先注意到这三个都是指针参数，那具体传了些啥东西进来呢？

* source是"ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"这个字符串
* str是我们输入的
* str1是最后出来的一个字符串
ok有了这些信息我们继续分析
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135688.png)
这里有个很明显的base加密过程
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135689.png)

`1.我们输入的字符是以8位二进制数，三个字符就是24位，base64即以6位为一组然后缺的补0。（其实做的时候还不太会base64甚至都是一个个字符还原的哈哈哈）`
但和一般的base64加密不太一样的事，它是拿我们输入的进行base64后去查表，然后拼成一个新的字符串。
有了这些信息我们继续往上看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135690.png)
这个函数basechange，跟进查看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135691.png)
原来是对原来的"ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"进行换表了。那我们的source就不在于是abcd。。
而是换表后的东西了。
所以逆向思路也有了。
先找到换完之后的表，然后根据GQTZlSqQXZ/ghxxwhju3hbuZ4wufWjujWrhYe7Rce7ju查看每个字符在表中的位置，然后再base64解密，就可以得到应该输入的字符串了。理论结束开始实践。
首先是找表啊，我不太想自己一个个输入了，就直接动态调试看一下了。
找到这个函数结束的位置4016E5，在这个地方下断点。
然后放进odg里面调试。注意这里面要我们输入字符串，我们必须输入33位的，要不然进入不了这个base64函数（当然可以在if上做点手脚）。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135692.png)
可以看到在这里我们找到了我要的表。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135693.png)
http://web.chacuo.net/netbasex
ok出现flag。
其实刚开始在做题的时候，base64不会，我就按照我的思路找到了每个字符在表中相对应的位置，以至于做的时候是把位置全换成二进制数
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135694.png)
然后高位补零4个为一组，找三个字符一个个还原的。哈哈哈有些麻烦了。
`2.补一个知识点吧，虽然不应该用到，但是记一下没问题。py中可以用bin（）直接输出二进制数`
#题目ez_XOR
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135695.png)
##解法
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135696.png)
无壳放ida
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135697.png)
逻辑还是很清晰的，我们输入的字符串进行处理后与str2进行比较，str2是明文被复制的。
那比较关键的对我们的str1干了什么了
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135698.png)
就是它函数XOR 还有一个参数是3
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135699.png)
也很简单啊上面依托不用看，是判断循环次数的。主要在于红框的处理。就是直接和9异或。
逆向思路：str2字符串自己与9异或
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135700.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135701.png)
有了
#题目easy_asm
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135702.png)
##解法
这道题是把我苦了很久的一道题，打也打不开（是msdos文件）。没办法直接扔进ida
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135703.png)
代码看上去是很轻松，但是不让f5，只能自己慢慢分析了。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135704.png)
这串代码是关键，把al当中的内容和$相比较，不相同则转移到前面，意思就是不相等就继续循环，找到al的内容为$就出去了。
所以我看别人的wp将这里其实就是判断到没到$，也就是说到$就到结尾了。
但是我看上面的东西我又不明白了，xor的结果不是应该存在al当中吗？
那后面的cmp中的al已经是被操作过的了，怎么会是输入的字符串呢？
输入后的字符串应该经过和0x10异或过了，应该是异或过的字符串，不应该啊。有明白的师傅可以评论指点一下。
。
。
那既然wp都这么讲，只能硬着头皮写了吧。
其实刚开始还有一个不明白，就是al到底是怎么存的。
直到看见这里的buffer
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135705.png)
实际上也是关系到键盘输入的。
。
。
那我们继续往下看，wp讲下面已经有密文了。
我往下看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135706.png)
wp有讲equal!前面的都是判断而已，不是密文，我也不大明白，只能硬着头皮看下去。
也就是说X。。后面的东西才是密文，把他经过^0x10。就是我们的flag。
贴上别的师傅的脚本
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020135707.png)
NSSCTF{Just_a_e3sy_aSm}
`3.总结一些知识吧，虽然这题没明白但还是学到了关于汇编的一些东西。。lodsb把si指向内容的数据给到al，然后根据df（方向）为决定增减si。。stosb把al的内容给di指向的地方，然后根据df增减di。。。xor把结果存在目标数，也就是前一个上面。。。。jnz当zf为0跳转也就是不相等时跳转。。。lea把后面的那个地址给前面的寄存器`
#总结
太菜了太菜了 最后那题明天再看看。
enbase64 base64查表换表
ez_XOR XOR
easy_asm asm 汇编