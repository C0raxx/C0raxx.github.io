---
layout:     post
title:      【NSSCTF刷题】《flowers》
subtitle:   CTF
date:       2023-12-13
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【CTF】
    - 【逆向】
    - 【NSSCTF】
---

# 题目 flowers

![image-20231214225802530](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312142316316.png)

## 解法

先exeinfo看看

![image-20231214225829806](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312142316318.png)

32位进去看看

既然是花指令，找找看吧

很明显是这个地方

![image-20231214230135565](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312142316319.png)

重新学了一下花指令，以下两篇博客给我很大的帮助

https://blog.csdn.net/demondev/article/details/91978869

https://blog.csdn.net/abel_big_xu/article/details/117927674

过两天再根据花指令专项发一个。



这里的思路就是看代码执行流程

401200后面的程序没法正确的反编译，由于都没有什么用，直接nop掉，指导里面的e8出，这是一个jmp

![image-20231214230439284](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312142316320.png)

就自动反汇编了，这时再去反编译

![image-20231214230607479](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312142316321.png)

代码就有了

![image-20231214230623345](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312142316323.png)

这是一个rc4的魔改版本，魔改在这里有个异或37

![image-20231214230648753](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312142316324.png)

上网找到个师傅的标准rc4解密脚本

```
import base64
def rc4_main(key = "init_key", message = "init_message"):
    print("RC4解密主函数调用成功")
    print('\n')
    s_box = rc4_init_sbox(key)
    crypt = rc4_excrypt(message, s_box)
    return crypt
    
def rc4_init_sbox(key):
    s_box = list(range(256)) 
    print("原来的 s 盒：%s" % s_box)
    print('\n')
    j = 0
    for i in range(256):
        j = (j + s_box[i] + ord(key[i % len(key)])) % 256
        s_box[i], s_box[j] = s_box[j], s_box[i]
    print("混乱后的 s 盒：%s"% s_box)
    print('\n')
    return s_box
    
def rc4_excrypt(plain, box):
    print("调用解密程序成功。")
    print('\n')
    plain = base64.b64decode(plain.encode('utf-8'))
    plain = bytes.decode(plain)
    res = []
    i = j = 0
    for s in plain:
        i = (i + 1) % 256
        j = (j + box[i]) % 256
        box[i], box[j] = box[j], box[i]
        t = (box[i] + box[j]) % 256
        k = box[t]
        res.append(chr(ord(s) ^ k))
    print("res用于解密字符串，解密后是：%res" %res)
    print('\n')
    cipher = "".join(res)
    print("解密后的字符串是：%s" %cipher)
    print('\n')
    print("解密后的输出(没经过任何编码):")
    print('\n')
    return  cipher

a=[0xc6,0x21,0xca,0xbf,0x51,0x43,0x37,0x31,0x75,0xe4,0x8e,0xc0,0x54,0x6f,0x8f,0xee,0xf8,0x5a,0xa2,0xc1,0xeb,0xa5,0x34,0x6d,0x71,0x55,0x8,0x7,0xb2,0xa8,0x2f,0xf4,0x51,0x8e,0xc,0xcc,0x33,0x53,0x31,0x0,0x40,0xd6,0xca,0xec,0xd4]
s=""
for i in a:
    s+=chr(i)

s=str(base64.b64encode(s.encode('utf-8')), 'utf-8')

rc4_main("Nu1Lctf233", s)


```

写的非常完备，我们魔改一下

![image-20231214231528156](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312142316325.png)

然后用这道题的数据

![image-20231214231547220](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312142316326.png)

完美