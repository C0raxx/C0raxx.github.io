---
layout:     post
title:      【NSSCTF刷题】《URL从哪儿来》
subtitle:   CTF
date:       2024-01-09
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【CTF】
    - 【逆向】
    - 【NSSCTF】
---

# 题目

![image-20240110211708298](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401102133150.png)



## 解法

最近正好在学恶意代码分析，就碰到这道题，就正好用恶意代码分析的方法来做这道题目。

下载下来是一个exe，我们exeinfo看看

![image-20240110211851902](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401102133151.png)

以为是 一个普通的exe，扔ida了

可以看到调用了好多api啊，也没啥实质性的跟flag相关的代码，用peid看导入函数

![image-20240110212246681](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401102133152.png)

可以看到有很多跟文件有关的api。

打开虚拟机放进火绒剑。



可以看到在c盘的临时文件夹里面创建出了一个临时的文件，我们找到那个文件

![image-20240110212546893](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401102133154.png)

![image-20240110212653245](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401102133155.png)

通过petools确定他是

![image-20240110212951010](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401102133156.png)

我们看看信息

![image-20240110213014459](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401102133157.png)

32位，进ida

进来看到一堆数组字符串，锁定一个关键函数进去看看

![image-20240110213043185](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401102133158.png)

有一个处理过程，应该不太难，直接在出口处下断

![image-20240110213208240](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401102133159.png)

跑起来之后发现，flag在eax的地址里面了

![image-20240110213313752](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401102133160.png)

`flag{6469616e-6369-626f-7169-746170617761}`

解毕