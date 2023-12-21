---
layout:     post
title:      【NSSCTF刷题】《IDA动态调试》《ezcode》《Char And Int》
subtitle:   CTF
date:       2023-12-20
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【CTF】
    - 【逆向】
    - 【NSSCTF】
---

# 题目

![image-20231221141927974](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312211501985.png)

## 解法

下载下来一个exe

![image-20231221141943607](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312211501987.png)

打开之后立马出来了

exeinfo看看信息

![image-20231221142011458](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312211501988.png)

ida看看

主函数很简单，但还是掉坑了。

本来以为东西就在printf打印了，所以直接在printf下断

![image-20231221142040109](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312211501989.png)

然后给我爆了一个错，我还以为是有啥反调试的

![image-20231221142200055](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312211501990.png)

没注意到这里这个函数

![image-20231221142215813](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312211501991.png)

进去之后如下

setflag,然后把flag存在的区域给delete掉，防止直接查看

![image-20231221142236400](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312211501992.png)

第一个 感觉是设置了什么密文

![image-20231221142318217](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312211501993.png)

第二个，看起来像一个解密过程。

![image-20231221142326306](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312211501994.png)

也不用ida了，用x64dbg。

在这个地方直接下断点

![image-20231221142426459](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312211501995.png)

然后根据上面存储的位置，直接找到flag

![image-20231221142443677](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312211501996.png)

# 题目 

![image-20231221143829421](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312211501997.png)

## 解法

这道题下载下来是一个py字节码题

看看

![image-20231221143908297](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312211501998.png)

主函数里面，四个子函数，然后给key赋值('XFFTnT')

调用func1 ，然后再encode 结果和ADkopgjJFP+28RYgXUxU2Oej比较

一点点看吧

首先是func1

func1里面调用了func2和func3，然后出现了sbox字样，可能是个rc4

![image-20231221144020867](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312211501999.png)

果然如此 func2标准的初始化sbox

![image-20231221144120158](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312211501000.png)

然后func3 只不过有些魔改

```
13           0 BUILD_LIST               0
              2 STORE_FAST               2 (res)

 14           4 LOAD_CONST               1 ('FSCTF')
              6 STORE_FAST               3 (y)

 15           8 LOAD_CONST               2 (0)
             10 DUP_TOP
             12 STORE_FAST               4 (i)
             14 STORE_FAST               5 (j)

 16          16 LOAD_FAST                0 (plain)
             18 GET_ITER
        >>   20 FOR_ITER                64 (to 150)
             22 STORE_FAST               6 (s)

 17          24 LOAD_FAST                4 (i)
             26 LOAD_CONST               3 (1)
             28 BINARY_ADD
             30 LOAD_CONST               4 (256)
             32 BINARY_MODULO
             34 STORE_FAST               4 (i)

 18          36 LOAD_FAST                5 (j)
             38 LOAD_FAST                1 (box)
             40 LOAD_FAST                4 (i)
             42 BINARY_SUBSCR
             44 BINARY_ADD
             46 LOAD_CONST               4 (256)
             48 BINARY_MODULO
             50 STORE_FAST               5 (j)

 19          52 LOAD_FAST                1 (box)
             54 LOAD_FAST                5 (j)
             56 BINARY_SUBSCR
             58 LOAD_FAST                1 (box)
             60 LOAD_FAST                4 (i)
             62 BINARY_SUBSCR
             64 ROT_TWO
             66 LOAD_FAST                1 (box)
             68 LOAD_FAST                4 (i)
             70 STORE_SUBSCR
             72 LOAD_FAST                1 (box)
             74 LOAD_FAST                5 (j)
             76 STORE_SUBSCR

 20          78 LOAD_FAST                1 (box)
             80 LOAD_FAST                4 (i)
             82 BINARY_SUBSCR
             84 LOAD_FAST                1 (box)
             86 LOAD_FAST                5 (j)
             88 BINARY_SUBSCR
             90 BINARY_ADD
             92 LOAD_CONST               4 (256)
             94 BINARY_MODULO
             96 STORE_FAST               7 (t)

 21          98 LOAD_FAST                1 (box)
            100 LOAD_FAST                7 (t)
            102 BINARY_SUBSCR
            104 STORE_FAST               8 (k)

 22         106 LOAD_FAST                2 (res)
            108 LOAD_METHOD              0 (append)
            110 LOAD_GLOBAL              1 (chr)
            112 LOAD_GLOBAL              2 (ord)
            114 LOAD_FAST                6 (s)
            116 CALL_FUNCTION            1
            118 LOAD_FAST                8 (k)
            120 BINARY_XOR
            122 LOAD_GLOBAL              2 (ord)
            124 LOAD_FAST                3 (y)
            126 LOAD_FAST                4 (i)
            128 LOAD_GLOBAL              3 (len)
            130 LOAD_FAST                3 (y)
            132 CALL_FUNCTION            1
            134 BINARY_MODULO
            136 BINARY_SUBSCR
            138 CALL_FUNCTION            1
            140 BINARY_XOR
            142 CALL_FUNCTION            1
            144 CALL_METHOD              1
            146 POP_TOP
            148 JUMP_ABSOLUTE           10 (to 20)

 23     >>  150 LOAD_CONST               5 ('')
            152 LOAD_METHOD              4 (join)
            154 LOAD_FAST                2 (res)
            156 CALL_METHOD              1
            158 STORE_FAST               9 (cipher)

 24         160 LOAD_FAST                9 (cipher)
            162 RETURN_VALUE
```

![image-20231221144211934](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312211501001.png)

用了fsctf这个key还做了一个异或操作

然后结果出来之后进行encode函数

![image-20231221144410793](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312211501002.png)

一个base64换表



所以逆向思路也有了 

先base64换表解码

然后根据结果进行rc4魔改解密



（差点以为做错了）

![image-20231221144447675](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312211501003.png)

写个脚本把他变成可见字符



[61, 46, 46, 35, 77, 216, 81, 239, 46, 242, 46, 116, 194, 208, 46, 118, 124, 183]

然后用下别人的脚本

```
def rc4_decrypt(ciphertext, key):
    # 初始化 S-box
    S = list(range(256))
    j = 0
    for i in range(256):
        j = (j + S[i] + key[i % len(key)]) % 256
        S[i], S[j] = S[j], S[i]

    # 初始化变量
    i = j = 0
    plaintext = []
    y = 'FSCTF'
    # 解密过程
    for byte in ciphertext:
        i = (i + 1) % 256
        j = (j + S[i]) % 256
        S[i], S[j] = S[j], S[i]
        k = S[(S[i] + S[j]) % 256]
        plaintext.append(byte ^ k ^ ord(y[i % 5]))

    return bytes(plaintext)


# 示例用法
encrypted_data = [61, 46, 7, 35, 77, 216, 81, 239, 157, 242, 12, 116, 194, 208, 173, 118, 124, 183]  # 替换成你的密文
encryption_key = b'XFFTnT'  # 替换成你的密钥

decrypted_data = rc4_decrypt(encrypted_data, encryption_key)
print("Decrypted Data:", decrypted_data.decode('utf-8'))

# Decrypted Data: FSCTF{G00d_j0b!!!}

```

出flag

# 题目

![image-20231221220615830](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312212212216.png)

## 解法

![image-20231221220640216](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312212212217.png)

输入个flag然后直接弹出去

放进ida64

主函数过程比较简单，大概就是这样

![image-20231221220735353](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312212212218.png)

进入NSS函数

可以发现逐个异或，然后结果和v3进行比较

![image-20231221220749133](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312212212219.png)

![image-20231221220925239](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312212212220.png)

刚开始没仔细看以为是flag是十一个字符，然后第一个异或0xa，第二个0xbb，以此类推。

但实际上他是这样子

比如我们输入NSSCTF{…},是一个四十四长度的字符串，然后第一轮读取CSSN（小端序）然后存进v4[0]，异或0xa,第二个一样倒序异或0xbb。

得此脚本

```
nEnc = [0x43535344, 0x667B46EF, 0x633635A9, 0x2D36BCBC, 0x626CDAD6, 0x65C8C8D2, 0x396BC3DC, 0x342DE9E4, 0x38396DFA,0x623737DD, 0x7D38393E]
nKey = [0xa , 0xbb , 0xccc , 0xdddd ,0xeeeee , 0xffffff, 0xeeeee, 0xdddd, 0xccc, 0xbb,0xa]
nTemp1 = []
for i in range(11):
    nTemp1.append(hex(nEnc[i] ^ nKey[i]))
#print(nTemp1)
#print(hex(a[0]^0xa))

#a = 1010

# 在之前的代码基础上，添加以下代码
def reverse_and_convert(string):
    # 去掉开头的"0x"
    string = string[2:]
    result = []
    
    # 逆序并两位两位输出字符
    for i in range(len(string)-1, -1, -2):
        # 取两位字符
        two_digits = string[max(i-1, 0):i+1]
        # 将字符转换为十六进制数字
        hex_num = int(two_digits, 16)
        # 将十六进制数字转换为字符串，加上"0x"前缀
        hex_str = hex(hex_num)

        # 存入数组
        result.append(hex_str)
    
    return result

# 测试例子
string = "0x12345678"
for i in nTemp1:
    result = reverse_and_convert(i)
    hex_nums = []

    for item in result:
    # 将每个字符串元素转换为十六进制数字
        hex_num = int(item, 16)
    # 存储到数组中
        hex_nums.append(hex_num)

    #print(hex_nums)
    for i in hex_nums:
        print(chr(i),end = '')
```

