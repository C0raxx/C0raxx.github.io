---
layout:     post
title:      【NSSCTF逆向】【2023题目】《easyAPK》《math》
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

#总览
easyapk 安卓逆向 java加密扩展 AES base58
math elf 五元一次方程 算法逆向
#题目 easyAPK
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136722.png)
##解法
第一次做有关安卓逆向的题目，没有工具没有思路，就结合别的师傅的wp来做题吧。
下载下来是一个apk，尝试了一些模拟器，没能成功，拿出以前的手机安装看看。
出来的是一个登录界面。
那大概就是需要账号和密码了，需要反编译apk了。
。
从别的师傅那里学的，安卓逆向需要一个叫做jadx的程序。详细的教程如下
https://blog.csdn.net/gqv2009/article/details/127323557
将jadx弄好之后，打开设这样的界面
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136723.png)
等于说右边是一些资源，节点啥的。
别的师傅讲这里可以通过xml找到主函数的界面。我这里是一个个一个点进去查看的
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136724.png)
这个login意思不就是登录嘛，感觉比较可疑，点进去还真是。
那就仔细看一下这个
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136725.png)
可以看到账号和密码分别就是str1和str2。
其中str1就是admin
str2还经过了一些加密操作，具体是什么操作
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136726.png)
可以看到这里的str2是通过plaintext传进来的。
而plaintext里面调用了下面的一个函数，是加密函数。里面那个java扩展加密是AES/CBC/PKCS5Padding
关于这个的知识如下
https://blog.csdn.net/wangmx1993328/article/details/106170060
其中iv是偏移量，密文是上面的base64加密后的东西（decode不是解密，相当于是取这个字符串吧），然后还有一个是key
我直接全文搜索了
找到了两处，结合wp锁定了一处
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136728.png)
ok，上解密
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136729.png)
到这里我们就有了密码了
登录进去是这样
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136730.png)
我也不太晓得是什么算法，就上这个网站
https://gchq.github.io/CyberChef/
里面打开hex和magic 相当于自己识别看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136731.png)
得到flag了
##总结

1. 安卓逆向工具jadx的使用
2. java加密扩展
3. AES CBC padding的方法
4. base58
#题目math
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136732.png)
##解法
下载下来是一个elf文件，放到linux执行看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136733.png)
大概是一个输入字符串然后处理再判断的题目。
直接上ida64
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136734.png)
从strings入手，找到关键字符串
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136735.png)
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136736.png)
关键信息在红框里面。
可以看到关键的加密里面有个savedregs
但是在上面我们输入的其实是v8，这里变成了savedregs，点进去看是待运算的，根据这个名字就是输入的字符串
也可以看到这里是五元一次方程，就按照下面写法（借鉴别人的wp）
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020136737.png)
出flag
##总结
1. elf文件
2. 五元一次方程写法
