---
layout:     post
title:      【NSSCTF刷题】《vm_wo》《Ez加密机》
subtitle:   CTF
date:       2024-01-16
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【CTF】
    - 【逆向】
    - 【NSSCTF】
---

# 题目

![image-20240117142454720](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401172215532.png)

## 解法

一道vm题

但是macos上面的，不好调试，就静态分析硬看吧，blue师傅用的qiling，还用不太来，本机ptython环境错乱了。

无壳64

![image-20240117165121206](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401172215533.png)

进ida

代码流程是很漂亮的，根据字符串长度先判断，然后进行虚拟机操作，操作之后再和密文比较。

![image-20240117165136379](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401172215534.png)

重点落在myoperate(__s, 29);这个函数里面，进去看看

函数主题赋了一堆值，然后调用interpretBytecode函数，进这个函数看看

```
switch ( v6 )
      {
        case 0:
          v11 = vm_body[v8];
          vm_body[v8] = vm_body[v10];
          vm_body[v10] = v11;
          goto LABEL_35;
        case 1:
          vm_body[v8] ^= vm_body[v9];
          goto LABEL_35;
        case 2:
          vm_body[v8] += v9;
          goto LABEL_35;
        case 3:
          vm_body[v8] += vm_body[v9];
          goto LABEL_35;
        case 4:
          vm_body[v8] -= v9;
          goto LABEL_35;
        case 5:
          vm_body[v8] -= vm_body[v9];
          goto LABEL_35;
        case 6:
          vm_body[v8] *= (_BYTE)v9;
          goto LABEL_35;
        case 7:
          vm_body[v8] *= vm_body[v9];
          goto LABEL_35;
        case 8:
          vm_body[v8] = (unsigned __int8)vm_body[v8] / v9;
          goto LABEL_35;
        case 9:
          vm_body[v8] = (unsigned __int8)vm_body[v8] / (unsigned __int8)vm_body[v9];
          goto LABEL_35;
        case 10:
          vm_body[v8] -= (unsigned __int8)vm_body[v8] / v9 * v9;
          goto LABEL_35;
        case 11:
          vm_body[v8] -= (unsigned __int8)vm_body[v8] / (unsigned __int8)vm_body[v9] * vm_body[v9];
          goto LABEL_35;
        case 12:
          v12 = (unsigned __int8)vm_body[v8];
          goto LABEL_18;
        case 13:
          v12 = (unsigned __int8)vm_body[0];
LABEL_18:
          vm_body[v8] = v12 << v9;
          goto LABEL_35;
        case 14:
          goto LABEL_28;
        case 15:
          v13 = (unsigned __int8)vm_body[v8];
          goto LABEL_21;
        case 16:
          v14 = *((int *)&_strlen + 52) - 1LL;
          *((_DWORD *)&_strlen + 52) = v14;
          v13 = vm_body[v14 + 16];
LABEL_21:
          result = printf("%d\n", v13);
          goto LABEL_35;
        case 17:
          if ( !vm_body[v8] )
            goto LABEL_25;
          goto LABEL_35;
        case 18:
          if ( vm_body[v8] )
LABEL_25:
            *((_DWORD *)&_strlen + 53) = v9;
          goto LABEL_35;
        case 19:
          *((_DWORD *)&_strlen + 53) = v7;
          goto LABEL_35;
        case 20:
          v8 = (unsigned __int8)vm_body[v8];
LABEL_28:
          v15 = vm_body[v8];
          goto LABEL_31;
        case 21:
          v16 = *((int *)&_strlen + 52) - 1LL;
          *((_DWORD *)&_strlen + 52) = v16;
          vm_body[0] = vm_body[v16 + 16];
          goto LABEL_35;
        case 22:
          v15 = v7;
LABEL_31:
          vm_body[(*((_DWORD *)&_strlen + 52))++ + 16] = v15;
          goto LABEL_35;
        case 23:
          goto LABEL_36;
        case 24:
          vm_body[0] = byte_100008002 | byte_100008001;
          goto LABEL_35;
        case 25:
          vm_body[v8] = (unsigned __int8)vm_body[0] >> v9;
          goto LABEL_35;
        case 26:
          vm_body[v8] = v9;
          goto LABEL_35;
```

一共26条opcode，还是很多，但不一定都要用到，看看是怎么个调用情况。

操作码就写在这些赋值code里面，而且是逆序存储的。

为1A.00.03.19.01.01.0D.02.07.18.01.02.01.00.03

![image-20240117170426515](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401172215535.png)

不过有一个我现在还没想通，为什么这里会把最前面的一个（也就是最后一个）覆盖掉

第一个最前面的2不算进去，直接从第二个02开始写，不太懂

![image-20240117170712876](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401172215536.png)

把opcode全写出来

```
opcode = [26, 0, 3, 25, 1, 1, 13, 2, 7, 24, 1, 2, 1, 0, 3, 26, 0, 3, 25,
          1, 2, 13, 2, 6, 24, 1, 2, 1, 0, 4, 26, 0, 3, 25, 1, 3, 13, 2, 5,
          24, 1, 2, 1, 0, 5, 26, 0, 3, 25, 1, 4, 13, 2, 4, 24, 1, 2, 1, 0, 6]
```

一共出现

```
26, 0, 3, 25, 1, 13, 2, 7, 24, 6, 4, 5]
```

那就还原这些个就可以了

0 swap a,b

![image-20240117170831056](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401172215537.png)

1 xor a,b

![image-20240117170844285](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401172215538.png)

2 add a, x(value)

![image-20240117170913130](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401172215539.png)

3 add a, b

![image-20240117170950944](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401172215540.png)

4 sub a, x(value)

![image-20240117171003918](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401172215541.png)

5 sub a, b

![image-20240117171023453](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401172215542.png)

6 7 mul

![image-20240117171656502](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401172215543.png)

13 左移

![image-20240117173257641](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401172215544.png)

24 a | b

![image-20240117172012613](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401172215545.png)

25 右移

![image-20240117172028196](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401172215546.png)

26 mov a，x（value）

![image-20240117172049970](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401172215547.png)

用别人的打印脚本

```
opcode = [26, 0, 3, 25, 1, 1, 13, 2, 7, 24, 1, 2, 1, 0, 3, 26, 0, 3, 25,
          1, 2, 13, 2, 6, 24, 1, 2, 1, 0, 4, 26, 0, 3, 25, 1, 3, 13, 2, 5,
          24, 1, 2, 1, 0, 5, 26, 0, 3, 25, 1, 4, 13, 2, 4, 24, 1, 2, 1, 0, 6]
# [26, 0, 3, 25, 1, 13, 2, 7, 24, 6, 4, 5]

i = 0
while opcode[i]:
    match opcode[i]:
        case 0:
            print(f"{i} swap reg%d reg%d" % (opcode[i+1],opcode[i+2]))
        case 1:
            print(f"{i} xor reg%d reg%d" % (opcode[i+1],opcode[i+2]))
        case 2:
            print(f"{i} add reg%d %d" % (opcode[i+1],opcode[i+2]))
        case 3:
            print(f"{i} add reg%d reg%d" % (opcode[i+1],opcode[i+2]))
        case 4:
            print(f"{i} sub reg%d %d" % (opcode[i+1],opcode[i+2]))
        case 5:
            print(f"{i} sub reg%d reg%d" % (opcode[i+1],opcode[i+2]))
        case 6:
            print(f"{i} mul reg%d %d" % (opcode[i+1],opcode[i+2]))
        case 7:
            print(f"{i} mul reg%d reg%d" % (opcode[i+1],opcode[i+2]))
        case 13:
            print(f"{i} mov reg%d reg0<<%d" % (opcode[i+1],opcode[i+2]))
        case 24:
            print(f"{i} reg0 = reg2 | reg1")
        case 25:
            print(f"{i} mov reg%d reg0>>%d" % (opcode[i+1],opcode[i+2]))
        case 26:
            print(f"{i} mov reg%d %d" % (opcode[i+1],opcode[i+2]))
    i += 3

```

可以得到 比较好看了

```
0 mov reg0 3
3 mov reg1 reg0>>1
6 mov reg2 reg0<<7
9 reg0 = reg2 | reg1
12 xor reg0 reg3
15 mov reg0 3
18 mov reg1 reg0>>2
21 mov reg2 reg0<<6
24 reg0 = reg2 | reg1
27 xor reg0 reg4
30 mov reg0 3
33 mov reg1 reg0>>3
36 mov reg2 reg0<<5
39 reg0 = reg2 | reg1
42 xor reg0 reg5
45 mov reg0 3
48 mov reg1 reg0>>4
51 mov reg2 reg0<<4
54 reg0 = reg2 | reg1
57 xor reg0 reg6
```

代码流程不复杂，单字节加密，reg3 到 reg6 为 (有点疑惑的)

![image-20240117173458014](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401172215548.png)

```
enc = [0xDF, 0xD5, 0xF1, 0xD1, 0xFF, 0xDB, 0xA1, 0xA5, 0x89, 0xBD, 0xE9, 0x95, 0xB3, 0x9D, 0xE9, 0xB3, 0x85, 0x99, 0x87,
       0xBF, 0xE9, 0xB1, 0x89, 0xE9, 0x91, 0x89, 0x89, 0x8F, 0xAD]
key = 0xBEEDBEEF.to_bytes(4, 'little')


def decode(s):
    s = s ^ key[3]
    s = (s << 4 | s >> 4) & 0xFF
    s = s ^ key[2]
    s = (s << 3 | s >> 5) & 0xFF
    s = s ^ key[1]
    s = (s << 2 | s >> 6) & 0xFF
    s = s ^ key[0]
    s = (s << 1 | s >> 7) & 0xFF
    return s


flag = []
for i in enc:
    i = ((i >> 3) | (i << 5) & 0xFF)
    flag.append(decode(i))
print(bytes(flag))
# b'DASCTF{you_are_right_so_cool}'

```

# 题目

![image-20240117220147630](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401172215549.png)

## 解法

一个exe

![image-20240117220337881](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401172215550.png)

直接上ida

根据string处找到关键字符串

![image-20240117220359592](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401172215551.png)

反编译 上面一大坨乱七八糟的函数

![image-20240117220759357](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401172215552.png)

在这里还有三个函数

sub_140003C30

![image-20240117220847145](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401172215553.png)

比较可疑的是传进去的a1111.. 后面动调看了一下把我们{}的内容替换成了1，然后进行长度比较

sub_1400036A0

base64 还有个换标操作 换表为`abcdefghijklmnopqrstuvwxyz0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ+/`

![image-20240117220946639](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401172215554.png)

sub_140003AE0

这个里面我是识别不出来，利用了signsrch识别了这个算法，是一个des

![image-20240117221310145](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401172215555.png)



最后要注意的是比较的这个字符串 实际上是做过手脚的，也就是不是下图这个str

![image-20240117221330299](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202401172215556.png)

动调得到

```python
0723105D5C12217DCDC3601F5ECB54DA9CCEC2279F1684A13A0D716D17217F4C9EA85FF1A42795731CA3C55D3A4D7BEA
```



好，用下别人的exp

```
from Crypto.Cipher import DES
import base64

encrypted_data = bytes.fromhex(
    "0723105D5C12217DCDC3601F5ECB54DA9CCEC2279F1684A13A0D716D17217F4C9EA85FF1A42795731CA3C55D3A4D7BEA")
print(encrypted_data)


def decrypt_DES(key, encrypted_data):
    cipher = DES.new(key, DES.MODE_ECB)
    decrypted_data = cipher.decrypt(encrypted_data)
    return decrypted_data


string1 = 'abcdefghijklmnopqrstuvwxyz0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ+/'
string2 = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/'

string = str.maketrans(string2, string1)

for i in range(1000000):
    key = base64.b64encode(str(i).zfill(6).encode()).decode().translate(string).encode()

    decrypted_data = decrypt_DES(key, encrypted_data)

    if b"DASCTF{" in decrypted_data:
        print(decrypted_data)
        exit()
# DASCTF{f771b96b71514bb6bc20f3275fa9404e}
```

