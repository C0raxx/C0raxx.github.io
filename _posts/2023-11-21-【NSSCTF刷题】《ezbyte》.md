---
layout:     post
title:      【NSSCTF刷题】《ezbyte》
subtitle:   CTF
date:       2023-11-21
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【CTF】
    - 【逆向】
    - 【NSSCTF】
---

## ezbyte

这是ciscn2023的一道re题，半年前我就在这道题目上坐过牢，昨天到今天研究了两天，总算把这道题以及其背后的机制了解了一下。让我们来好好做一下这道题。

题目拿到

![image-20231122184943218](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311221948462.png)

exeinfo一下

64位elf

![image-20231122185204729](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311221948463.png)

先放到linux里面泡泡试试

输入123456之后回显了一遍，然后没有别的字符串了。



![image-20231122190025172](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311221948464.png)

拿ida简单看看

根据string给的一点点信息

![image-20231122185401116](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311221948465.png)

我们找到这里

这里只有100的字样，初步猜测是有处理把yes隐藏掉了



![image-20231122185415252](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311221948467.png)

但至少这里也算进入题目主体部分了吧。

格式化了一个什么字符串，先给他一个scanf

![image-20231122190133829](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311221948468.png)

后面两条看不清楚要干啥

在后面到这个函数里面

![image-20231122190323861](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311221948469.png)

和什么比较判断呢？

![image-20231122190409941](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311221948470.png)

看着好像是输入的东西和正确flag作某部分比较，如果全对进入这里

![image-20231122190457165](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311221948471.png)

好像有点不对

说实话我也不知道为啥是这样，看着也不像是花指令。

![image-20231122190521700](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311221948472.png)





做到这里我就卡住了。只能搜索学习各位师傅的思路。

根据

这个函数的位置 404dc1

![image-20231122192133001](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311221948473.png)

找到yes字样，在这里，前面有一个比较加跳转

而看了前面几个函数，均没有对r12进行操作，可以猜测我们想要分析的操作被隐藏起来了。

![image-20231122192218962](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311221948474.png)

。

那到底是怎么异常起来的呢？

是这个里面，进行格式比对之后执行的函数，我们点进去



![image-20231122192612499](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311221948475.png)

不知道为什么，我这里没有符号文件

![image-20231122192903885](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311221948476.png)

别人的长这样

![image-20231122192916752](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311221948477.png)

涉及到异常处理。

这道题目的异常处理是这个意思：

简单来讲，就是发生异常的时候，程序会随着调用链不断往上进行栈展开（栈回溯），知道找到能够解决这个异常的catch块。

而在进行栈回溯时，必定涉及到如何进行恢复上个调用程序的一些变量（包括寄存器，栈等等）

在这之间有个DWARF Expression，因为进行恢复环境的时候本来的cfi用起来不方便，之后引入的DWARF Expression，算是用他的表达式以一个更加方便的方式进行辅助栈展开。甚至可以把他看做成一个**虚拟机**。

而本体的操作就是被隐藏在这个栈展开中DWARF的“**虚拟机**”字节码中。







在linux当中执行这行指令，把所需要的WARF调试信息打印出来。

![image-20231122193713012](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311221948478.png)

有很多的内容，而其中的这里是比较反常的。

![image-20231122193811396](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311221948479.png)

```
DW_CFA_advance_loc: 5 to 0000000000404bfa
  DW_CFA_def_cfa_offset: 16
  DW_CFA_offset: r6 (rbp) at cfa-16
  DW_CFA_advance_loc: 3 to 0000000000404bfd
  DW_CFA_def_cfa_register: r6 (rbp)
  DW_CFA_val_expression: r12 (r12) (DW_OP_constu: 2616514329260088143; DW_OP_constu: 1237891274917891239; DW_OP_constu: 1892739; DW_OP_breg12 (r12): 0; DW_OP_plus; DW_OP_xor; DW_OP_xor; DW_OP_constu: 8502251781212277489; DW_OP_constu: 1209847170981118947; DW_OP_constu: 8971237; DW_OP_breg13 (r13): 0; DW_OP_plus; DW_OP_xor; DW_OP_xor; DW_OP_or; DW_OP_constu: 2451795628338718684; DW_OP_constu: 1098791727398412397; DW_OP_constu: 1512312; DW_OP_breg14 (r14): 0; DW_OP_plus; DW_OP_xor; DW_OP_xor; DW_OP_or; DW_OP_constu: 8722213363631027234; DW_OP_constu: 1890878197237214971; DW_OP_constu: 9123704; DW_OP_breg15 (r15): 0; DW_OP_plus; DW_OP_xor; DW_OP_xor; DW_OP_or)
  DW_CFA_nop
  DW_CFA_nop
  DW_CFA_nop
 
```

我对这个DWARF也不太了解，放进AI里面看看

![image-20231122194324787](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311221948480.png)

如果能看到这里的话，逻辑其实也不难，从后往前看（回溯过程，从后往前，而且可以看到r12是最前面的，而正向流程一个是r12最后。

我们的输入和9123704 相加，然后异或出去

用了别人的exp

```
def decrypt():
    r15 = (8722213363631027234 ^ 1890878197237214971) - 9123704
    r14 = (2451795628338718684 ^ 1098791727398412397) - 1512312
    r13 = (8502251781212277489 ^ 1209847170981118947) - 8971237
    r12 = (2616514329260088143 ^ 1237891274917891239) - 1892739
    print(hex(r12))
    print(hex(r13))
    print(hex(r14))
    print(hex(r15))
    import binascii
    hexstring = "65363039656662352d653730652d346539342d616336392d6163333164393663"
    print("flag{" + binascii.unhexlify(hexstring).decode(encoding="utf-8") + "3861}")
    # print(binascii.unhexlify(hex(r15)[2:]).decode('utf-8'), end='')
    # print(binascii.unhexlify(hex(r14)[2:]).decode('utf-8'), end='')
    # print(binascii.unhexlify(hex(r13)[2:]).decode('utf-8'), end=' ')
    # print(binascii.unhexlify(hex(r12)[2:]).decode('utf-8'), end=' ')

decrypt()


```

**flag{e609efb5-e70e-4e94-ac69-ac31d96c3861}**



本文思路取自这个师傅

[2023 CTF国赛初赛（CISCN） re- ez_byte_二木先生啊的博客-CSDN博客](https://blog.csdn.net/qq_54894802/article/details/130956092)