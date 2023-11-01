---
layout:     post
title:      【NSSCTF逆向】《VidarCamera》《shellcode》
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

#题目 VidarCamera
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020133294.png)
##解法
这是一道安卓逆向题目，放在模拟器里打开看看
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020133296.png)
需要输入一个序列号啥的，扔jadx里吧。
通过字符串搜索定位到关键代码
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020133297.png)
这里应该就是一个变种TEA，更改了加密轮次，delta。
不过是TEA加密，写脚本不太难，自己的太丑了，贴个别人的

<details>
<summary>点击查看代码</summary>

```
from Crypto.Util.number import *

def decrypt(v, k):
    v0 = v[0]
    v1 = v[1]
    delta = 0x34566543
    x = delta * 33
    for i in range(33):
        x -= delta
        x = x & 0xFFFFFFFF
        v1 -= (((v0 << 4) ^ (v0 >> 5)) + v0) ^ (x + k[(x >> 11) & 3])
        v1 = v1 & 0xFFFFFFFF
        v0 -= (((v1 << 4) ^ (v1 >> 5)) + v1) ^ (x + k[x & 3]) ^ x
        v0 = v0 & 0xFFFFFFFF
    v[0] = v0
    v[1] = v1
    return v
    
c = [0x260202FA, 0x1B451064, 0x867B61F1, 0x228033C5, 0xF15D82DC, 0x9D8430B1, 0x19F2B1E7, 0x2BBA859C, 0x2A08291D, 0xDC707918]
key = [2233, 4455, 6677, 8899]
flag = b''
for i in range(len(c)-1):
    d = decrypt(c[-2:], key)
    flag = long_to_bytes(d[1])[::-1] + flag
    c = c[:-2] + [d[0]]

print(flag)
```
</details>

![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020133298.png)

出flag
#题目 shellcode
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020133299.png)
##解法
pwn涉及的比较少，题目有一些细节没有明白，但草草还是可以做一下
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020133300.png)
查壳信息
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020133301.png)
进入main函数，可以看到有个类似base64的东东
![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020133302.png)
得到数据
然后用python写一个脚本用来解密然后输出成一个二进制文件
<details>
<summary>点击查看代码</summary>

```
import base64

base64_string = "YOUR_BASE64_ENCODED_STRING"
output_file = "output.bin"

# 解码 Base64 字符串
decoded_data = base64.b64decode(base64_string)

# 将解码后的数据写入二进制文件
with open(output_file, "wb") as file:
    file.write(decoded_data)

print("解码成功，并已保存为二进制文件:", output_file)
```
</details>

ok，出来一个bin文件，扔进ida，这里需要create a function才能反编译

![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020133303.png)

![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020133304.png)
很明显的一个tea操作
写脚本

![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020133305.png)

出flag

![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311020133306.png)
