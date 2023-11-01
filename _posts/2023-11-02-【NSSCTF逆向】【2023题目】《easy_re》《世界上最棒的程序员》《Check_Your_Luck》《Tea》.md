---
layout:     post
title:      【NSSCTF逆向】《easy_re》《世界上最棒的程序员》《Check_Your_Luck》《Tea》
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

#题目easy_re
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136818.png)
##解法
很简单的一道题
考的就是upx脱壳 base64加解密
拿到文件
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136819.png)
upx壳 upx -d 脱壳
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136820.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136821.png)
无壳
放进ida
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136822.png)
很明显关键在于这个判断的两个字符串是啥。现在我们看看我们输入的s变成了什么。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136823.png)
进入func
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136824.png)
func的内容主要是对s进行操作然后给encode_
这次我看明白了，这个很明显是个base64。
我们再出去看a是什么
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136825.png)
a被base64了。
直接点a看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136826.png)
直接解密
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136827.png)
出flag
#题目世界上最棒的程序员
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136828.png)
##解法
这道题在比赛时候做的，也是一道签到题，直接扔进ida里看strings窗口就可以了
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136829.png)
#题目Check_Your_Luck
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136830.png)
##解法
也是一道很简单的签到题，考的就是五元一次方程组的解
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136831.png)
笨办法，直接找在线的计算器
https://www.buildenvi.com/gongju/formula/3lck4
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136832.png)
计算出flag
#题目Tea
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136833.png)
##解法
这道题算是我做的的第一道tea题目。拿到题目先exeinfo一下
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136834.png)
ok这道题没壳，放进ida里面，是start函数，暂时没有入口，我们直接找关键字符串，定位到这个关键函数位置
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136835.png)
ok，先学习一下tea的知识吧
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136836.png)
`1.是一个新知识啊，简单来讲待加密数据是两个2个32位，秘钥是4个32位。关键的过程也就如上图所示（为一轮的过程，在这一轮过程中v1数据由三个表达式拼接，分别是k【0】<<4,DELTA,K[1]>>5三者异或组成，下面的也是同样的原理，然后32轮或者64轮得到最终加密的数据。而这个算法也是可逆的，可以完全将加密过程逆转，加减相倒即可。）`
ok，大概就是tea的主题思路。
我们找一找这个tea加密函数具体在哪。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136838.png)
ok在这里，在这里看的时候其实刚开始迷惑的是这个4i64怎么这么多，后面看到其实把四位进行一个处理也是可以的。
更让我感到奇怪的是![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136839.png)
16*到底是什么意思。
后面看了一下别人的博客才明白是我对左移右移还欠缺理解。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136840.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136841.png)

`左移右移用简单的算术运算实际上是乘以（除以）几个2。32 >> 2  = 32 / 2^2  = 8

17 >> 2 = 17 / 2^2 = 4.25 = 4（因为只取整数部分）

512 >> 10 = 512 / 2^10 = 0.5 = 0（因为只取整数部分）`
所以这个16*，如果以四位为一组，其实也就是每个左移了四位的意思。
ok，逆向思路已有，写脚本（脚本没存贴别的师傅的wp了）
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136842.png)
#总结
easy_re upx脱壳   base64
世界上最棒的程序员 字符串查找
Check_Your_Luck  线性方程组求解
Tea tea          算法