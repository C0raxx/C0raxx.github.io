---
layout:     post
title:      【NSSCTF逆向】《snake》《easy_asm》
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

#题目snake
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134831.png)
##解法
是一个pyc文件，经过之前的学习。了解到可以通过uncompyle6的反编译还原出一个py脚本
试试
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134832.png)
反编译不出来，后面查阅了别人的资料了解到这个魔数是类似于标记python脚本的东西，uncompyle6的报错告诉我们没找到这个魔数
于是winhex看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134833.png)
结合看别人的正常的pyc文件头，看到这里的头已经被修改过了。
`所以我们现在要做的就是还原出一个好的magic number然后让他能被反编译。`
起先我认为，这个是根据本机的环境，也就是python来决定魔数的，于是我用我电脑上的一个py文件生成了一个pyc文件
然后可以看到这个magic number
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134834.png)
可以看到这是61 0D 0D 0A,然后我填的题目里进行反编译
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134835.png)
后来试了一下python3.7才明白，这个number得根据编译文件的版本，题目可以看到
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134836.png)
再网上找了37的魔数 https://www.cnblogs.com/Here-is-SG/p/15885799.html
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134837.png)
（得倒置）
再翻译就成功了，然后就有了一个py脚本
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134838.png)
关键在于这里，直接截取关键段并输出
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134839.png)
LitCTF{python_snake_is_so_easy!}
#题目easy_asm
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134840.png)
##解法
这道题是把我苦了很久的一道题，打也打不开（是msdos文件）。没办法直接扔进ida
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134841.png)
代码看上去是很轻松，但是不让f5，只能自己慢慢分析了。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134842.png)
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
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134843.png)
实际上也是关系到键盘输入的。
。
。
那我们继续往下看，wp讲下面已经有密文了。
我往下看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134844.png)
wp有讲equal!前面的都是判断而已，不是密文，我也不大明白，只能硬着头皮看下去。
也就是说X。。后面的东西才是密文，把他经过^0x10。就是我们的flag。
贴上别的师傅的脚本
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020134845.png)
NSSCTF{Just_a_e3sy_aSm}

附：其实这题前几天已经做了，但是因为想不明白最后的那个判断过程，所以也没交flag。
现在和别的师傅交流了一下，探讨了一下这道题中读取字符和比较的详细过程，虽然现在暂定的结果还是题有些问题，但还是学了不少。
。
。今天打了三个多小时羽毛球。爽的一。题做的有点少了最近也在考试。加大力度

#总结
snake pyc反编译 反混淆
easy_asm 二进制 asm