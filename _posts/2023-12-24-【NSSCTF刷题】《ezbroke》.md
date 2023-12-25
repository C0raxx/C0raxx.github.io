---
layout:     post
title:      【NSSCTF刷题】《ezbroke》
subtitle:   CTF
date:       2023-12-24
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【CTF】
    - 【逆向】
    - 【NSSCTF】
---

# ezbroke

![image-20231225223655382](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312252253845.png)

## 解法

下载下来 一个exe，点开看看

![image-20231225223723880](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312252253846.png)

无法运行，exeinfo看看

长这样，可能损坏了，winhex看看

![image-20231225223737725](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312252253847.png)

发现dos头和pe的文件偏移都有问题，修改。

![image-20231225224125854](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312252253848.png)

发现可以正常打开了

![image-20231225224134863](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312252253849.png)

但是发现仍然有upx壳

![image-20231225224151831](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312252253850.png)

脱壳，发现失败了

![image-20231225224227431](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312252253851.png)

继续winhex看看，发现都修改了

![image-20231225224256155](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312252253852.png)

修改后

![image-20231225224322610](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312252253853.png)

再去脱壳，成功

![image-20231225224349960](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312252253854.png)

打开ida32，发现好像是个VM题奥

![image-20231225224636586](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312252253855.png)

不过还是比较友好，直接给了我有啥操作

![image-20231225224658558](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312252253856.png)

一个一个看过之后，应该就是这了，异或0x17

![image-20231225224718995](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312252253857.png)

然后进check比较

![image-20231225224736726](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312252253859.png)

所以逆向思路就是，把encflag先dump出来，然后再异或回去

```
Enc = [
  0x51, 0x44, 0x54, 0x43, 0x51, 0x6C, 0x4E, 0x27, 0x62, 0x37, 
  0x64, 0x62, 0x74, 0x74, 0x72, 0x64, 0x64, 0x71, 0x62, 0x26, 
  0x26, 0x6E, 0x37, 0x75, 0x65, 0x27, 0x7C, 0x24, 0x37, 0x7A, 
  0x6E, 0x37, 0x67, 0x65, 0x27, 0x63, 0x24, 0x74, 0x63, 0x26, 
  0x27, 0x79, 0x36, 0x36, 0x36, 0x6A, 0x00
]
cFlag = ""

for i in range(len(Enc)):
    cFlag += chr(Enc[i] ^ 0x17)

print(cFlag)
```

![image-20231225225014018](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312252253860.png)

出了

![image-20231225225031473](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312252253861.png)