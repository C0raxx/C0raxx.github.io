---
layout:     post
title:      【160Crackme】《Cabeca》
subtitle:   Crackme
date:       2023-11-14
author:     Corax
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 【Crackme】
    - 【逆向】
---

# Cabeca

这个crackme吧。。能学到一些东西，但是要写keygen是比较恶心的。

## 解法

查壳

无壳32位，上OD

![image-20231115115819130](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311151224332.png)

定位关键字符串

borland写的程序，关键字符串好像都在下面

![image-20231115120015379](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311151224333.png)

定位过来

这题和最近做的不太一样，好像错误回显不在正确回显的附近上下找找。

![image-20231115120201424](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311151224334.png)

找到错误回显位置 和关键跳地址

多个地方跳到这个D4E5，跳过了正确回显call

![image-20231115120352157](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311151224335.png)

![image-20231115120326774](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311151224336.png)

### 爆破

先爆破一下吧

在e5位置汇编成如下，这样就会直接进入正确回显的流程

![image-20231115120544486](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311151224338.png)



爆破成功

![image-20231115120620584](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311151224339.png)

### 算法逆向

这个算法流程稍微有点长，我现在开始的地方下断了

![image-20231115120908302](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311151224340.png)



两个可疑的比较

可以在这个函数在我们输入字符串之后下断，并且这两个地址分别是8f5f 320，在往下跑

![image-20231115120952358](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311151224341.png)

计算器算一下是36703 800



往下跑

经过一个函数已经变成了具体的数字了

![image-20231115121352596](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311151224342.png)

然后比较跳入错误回显

![image-20231115121426579](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311151224343.png)



于是就想试试，难道两个serial就是36703和800吗

成功了，但我感觉这道题应该没有那么简单吧。换个name

![image-20231115121511383](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311151224344.png)

错误

![image-20231115121544717](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311151224345.png)





懂了，原来是根据name生成的serial，编码在内存地址里面。

我走了一段歪路，其实只要在那两个地址下内存写入断点就可以了。

要在还没跑的时候 下断

跳到这里了，应该是有个索引在的，我们根据堆栈信息

![image-20231115121935927](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311151224346.png)

来到这里

![image-20231115122022708](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311151224347.png)

这里的作用就是检验是不是字母（因为8+72）就到字母边界了。

然后根据字母所在的位置 *4（因为数据长度），简单来讲，就是比如我们输入c，就是3，找到case63，加上对应的值就可以了。

把这个东西复制进excel，变成可读的样子，然后直接找对应的字符累加就可以了。

![image-20231115122211096](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311151224348.png)





我们试一个name hyyyy

h 750 b

y 2d13 8

就是750 + 2d13*4

b+ 8*4

![image-20231115122410126](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311151224349.png)

![image-20231115122420278](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311151224350.png)



成功

![image-20231115122429832](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311151224351.png)