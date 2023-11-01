---
layout:     post
title:      【攻防世界逆向】【高手题】《re1-100》《EasyRE》《parallel-comparator-200》《simple-check-100》
subtitle:   Crackme
date:       2023-09-30
author:     Corax
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 【CTF】
    - 【逆向】
    - 【攻防世界】
---

#题目re-100
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111197.png)
##解法
exeinfo![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111198.png)
无壳64位
放进ida64
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111199.png)
上面很多内容，比较冗长，下面这些判断比较吸睛。大概就是进行一些对比后最后和一个字符串进行比较。直接提交，失败的。那我看一下上面的这个函数做了什么。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111200.png)
是将这个字符串打乱了，打乱了之后才是下面那个。所以我们将原字符串10位分一个，然后重组一年，上面加密过程是3412 还原即可，出现flag
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111201.png)
（第二行少了个0，要加上）
#题目EasyRE
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111202.png)
##解法
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111203.png)
无壳exe扔进ida
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111204.png)
查看伪代码之后从下往上看，定位到这里![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111205.png)
最后是与这个字符串作比较，于是可以想到这道题是把输入的字符串作某些处理，再比较判断对错。现在要看这个字符串传到了那里去。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111206.png)
点进去又出现scanf这里大概就是把输入的字符串传入这个数组。但是问题来了
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111207.png)
在理应是处理函数的地方确没有出现这个数组，这种情况我第一次遇到。
借鉴了一下别人的思路，要用到地址的知识了。
可以看看这个处理过程具体是怎么做的。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111208.png)
下面那个for比较简单。主要是看上面的过程。v3作计数器，是循环24次。对应着我们输入的24位的字符串。
拿这个v4是什么，v4和v10有关。取v10的地址然后加7？这是什么意思。
可以看到ida帮忙找到了这些地址。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111210.png)
arglist是16位。应该从ebp-36（24h）往上存存到ebp-20也就存完了，这里是16位。但是我们输入的是24位啊！原来如此，剩下的八位正好从20起也就是ebp-14h开始存，往上到12存完（12是没有的 13有内容）。那么v4就是我们输入的最后一个字符了！那么下面是什么意思也就明白了
v5取了最后一个字符，然后v4往前，把v5赋给后面的的那串字符应该在的位置。也就是倒置。
所以写脚本的思路就是先把最后的字符串完成最后for循环当中的逆向算法，逆出来一个字符串，再倒置。
还有个东西其实我有疏漏啊。最后的字符串是18位的，应该是反编译的时候有啥问题，最好的办法是找到地质区。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111211.png)
选中shift e，可以直接复制了。出来倒置的flag![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111212.png)
逆一下就可以了。
#题目parallel-comparator-200
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111213.png)
##解法
这题真是把我搞崩溃了，看别人的writeup都看了好久学了好多。开始吧。
下载下来是一个C源文件，那好我们直接用vs打开它。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111214.png)
打开出来的东西很不友善啊，没事，我们先找判断。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111215.png)
可以看出我们需要isok为1，也就是这个函数的返回值为1.我们往上看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111216.png)
可以看到要为1就不能走这条if判断语句，所以generated_string[i] = just_a_string[i]这俩是相等的。再往上看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111217.png)
可以看到这里对generated_string做了一个加法，加上result，但后面判断语句看到了这俩货必须相等。所以result只能为0。result又是咋操作的。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111218.png)
可以看到我这边划得东西很多，因为这里的知识量很大。里面涉及pthrea的一个操作，先看上面的
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111219.png)
第一个参数是用来表示一个线程，第二个手动设置线程的属性，第三个我们穿的参数进去要调用什么函数，第四个使我们穿的参数。其中细节就是第三个，我直接上图
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111220.png)
重点在于以函数指针的方式表达要传的的函数，也就是说在函数中我们做的具体操作是可以被接受的。好我们看下面的。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111221.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111222.png)
写的也很明了了，用result存上面那个创建线程函数的返回值，一个发起一个接受，现在具体就在于上面的那个checking到底做了什么？
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111223.png)
好，我们需要的是result等于0
(argument[0]+argument[1]) ^ argument[2]也就是这个等于0，所以argument[0]+argument[1]=argument[2]。其中argument[2]是我门输入的，用来解答案的，不用管。那前者两个就很关键，他聊到底是啥。
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111224.png)
可以看到argument[1]是这个字符串当中一个一个的值，checking是外套了一个循环的，所以每个都用到了，那这个firstletter是啥？
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111225.png)
要对一些数字比较敏感，其中97asc2中就是a那么firstletter就是从b开始（前面有initialization_number % 26必不为0），好。
思路就是：对从b开始到z（因为initialization_number % 26必然小于等于25，正好到z），对differences中的数字一个个加。就可以得到答案了！
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111226.png)
#题目simple-check-100
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020111227.png)
##解法
每天在写 还在研究