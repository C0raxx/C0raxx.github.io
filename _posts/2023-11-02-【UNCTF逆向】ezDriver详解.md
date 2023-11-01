---
layout:     post
title:      【UNCTF逆向】ezDriver详解
subtitle:   CTF
date:       2023-09-30
author:     Corax
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 【CTF】
    - 【逆向】
    - 【UNCTF】
---

#题目ezDriver
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121077.png)
##解法
一拿到题是一个sys文件，有点蒙，拿exeinfo打开看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121078.png)
没看懂，扔到ida64里面看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121079.png)
没有main函数，一个个看了一下，发现了一个可疑的函数
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121080.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121081.png)
不晓得这个是啥意思，后面查了一下别人的writeup发现这个是tea加密，还是个变种tea加密，无奈本人加解密知识不精，直接就借鉴别人的题解吧（侵删）
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121082.png)
得到flag是一串一六进制数
74636E756F447B66756F795F6E61775F6F745F74635F615F6F5F707565745F667D3F610转换一下
tcnuoD{fuoy_naw_ot_tc_a_o_puet_f}?a，
四个字符转换 得到unctf{Do_you_want_to_a_cup_of_tea?}
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020121083.png)
