---
layout:     post
title:      【NSSCTF刷题】《Blast》《Redirect》
subtitle:   CTF
date:       2023-12-18
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【CTF】
    - 【逆向】
    - 【NSSCTF】
---

# 题目Blast

![image-20231219152225942](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312192336009.png)

## 解法

下载下来是一个elf，放进ida

查看字符串

![image-20231219152350988](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312192336011.png)

到这个地方

![image-20231219152425294](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312192336012.png)

可以看到一个很明显的判断条件

v8是输入的进行操作4

s2点进去

一大长串

![image-20231219152459421](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312192336013.png)

拿几个来试试

![image-20231219152549399](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312192336014.png)

看到这几个md5(md5()),知道这玩意应该是个双重md5

写脚本

```
from hashlib import *

cmp=['14d89c38cd0fb23a14be2798d449c182', 'a94837b18f8f43f29448b40a6e7386ba', 'af85d512594fc84a5c65ec9970956ea5', 'af85d512594fc84a5c65ec9970956ea5', '10e21da237a4a1491e769df6f4c3b419', 'a705e8280082f93f07e3486636f3827a', '297e7ca127d2eef674c119331fe30dff', 'b5d2099e49bdb07b8176dff5e23b3c14', '83be264eb452fcf0a1c322f2c7cbf987', 'a94837b18f8f43f29448b40a6e7386ba', '71b0438bf46aa26928c7f5a371d619e1', 'a705e8280082f93f07e3486636f3827a', 'ac49073a7165f41c57eb2c1806a7092e', 'a94837b18f8f43f29448b40a6e7386ba', 'af85d512594fc84a5c65ec9970956ea5', 'ed108f6919ebadc8e809f8b86ef40b05', '10e21da237a4a1491e769df6f4c3b419', '3cfd436919bc3107d68b912ee647f341', 'a705e8280082f93f07e3486636f3827a', '65c162f7c43612ba1bdf4d0f2912bbc0', '10e21da237a4a1491e769df6f4c3b419', 'a705e8280082f93f07e3486636f3827a', '3cfd436919bc3107d68b912ee647f341', '557460d317ae874c924e9be336a83cbe', 'a705e8280082f93f07e3486636f3827a', '9203d8a26e241e63e4b35b3527440998', '10e21da237a4a1491e769df6f4c3b419', 'f91b2663febba8a884487f7de5e1d249', 'a705e8280082f93f07e3486636f3827a', 'd7afde3e7059cd0a0fe09eec4b0008cd', '488c428cd4a8d916deee7c1613c8b2fd', '39abe4bca904bca5a11121955a2996bf', 'a705e8280082f93f07e3486636f3827a', '3cfd436919bc3107d68b912ee647f341', '39abe4bca904bca5a11121955a2996bf', '4e44f1ac85cd60e3caa56bfd4afb675e', '45cf8ddfae1d78741d8f1c622689e4af', '3cfd436919bc3107d68b912ee647f341', '39abe4bca904bca5a11121955a2996bf', '4e44f1ac85cd60e3caa56bfd4afb675e', '37327bb06c83cb29cefde1963ea588aa', 'a705e8280082f93f07e3486636f3827a', '23e65a679105b85c5dc7034fded4fb5f', '10e21da237a4a1491e769df6f4c3b419', '71b0438bf46aa26928c7f5a371d619e1', 'af85d512594fc84a5c65ec9970956ea5', '39abe4bca904bca5a11121955a2996bf']
table=[]
table_md5=[]

for i in range(33,127):
    table.append(chr(i))
    a = (md5(chr(i).encode()).hexdigest()).encode()
    print(chr(i))
    print(a)
    b = md5((md5(chr(i).encode()).hexdigest()).encode()).hexdigest()
    print(b)
    table_md5.append(b)
#print(table)
#print(table_md5)
for i in range(len(cmp)):
    print(table[table_md5.index(cmp[i])],end='')#Hello_Ctfer_Velcom_To_my_Mov_and_md5(md5)_world
00
```

出flag

# 题目Redirect

![image-20231219232745957](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312192336015.png)

## 解法

还是比较一个有新意的题目

![image-20231219232951195](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312192336016.png)

让我们输入NULL,但很明显这个null应该是空值,尝试输入space,还会要求输入,应该是做了过滤,没事我们反编译试试.

![image-20231219233242612](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312192336017.png)

ida进去之后根据strings定位

根据正确信息来到这里

![image-20231219233330071](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312192336018.png)

```
__int64 __fastcall NextStage(unsigned __int8 *a1, unsigned __int64 a2)
{
  __int64 v2; // rax
  __int64 v3; // rax
  __int64 v4; // rax
  char v6[11]; // [rsp+25h] [rbp-5Bh] BYREF
  _QWORD v7[9]; // [rsp+30h] [rbp-50h] BYREF
  int v8; // [rsp+7Ch] [rbp-4h]

  v7[0] = 0xA1308C516A28114FLL;
  v7[1] = 0x78DE79E973A3AD1DLL;
  v7[2] = 0x686E5018101DB32FLL;
  v7[3] = 0xDB9C8282515A206ALL;
  v7[4] = 0x680BD34CA4EEA7E1LL;
  v7[5] = 0x1BE376DF6C2BD8E6LL;
  v7[6] = 0x83424F474BCE8952LL;
  v7[7] = 0xE185321994B69B72LL;
  strcpy(v6, "nSsCtf2023");
  v8 = 10;
  rc4_crypt(a1, (unsigned __int8 *)a2, (unsigned __int8 *)0x40, (__int64)v7, (unsigned __int64)v6);
  std::operator<<<std::char_traits<char>>(a1, a2, "Oh! Your input is ", refptr__ZSt4cout);
  GetStdHandle((DWORD)a1);
  SetConsoleTextAttribute(a1, a2);
  std::operator<<<std::char_traits<char>>(a1, a2, "NULL", refptr__ZSt4cout);
  GetStdHandle((DWORD)a1);
  SetConsoleTextAttribute(a1, a2);
  v2 = std::operator<<<std::char_traits<char>>(a1, a2, "!", refptr__ZSt4cout);
  std::ostream::operator<<(a1, a2, refptr__ZSt4endlIcSt11char_traitsIcEERSt13basic_ostreamIT_T0_ES6_, v2);
  v3 = std::operator<<<std::char_traits<char>>(a1, a2, "The flag is: ", refptr__ZSt4cout);
  v4 = std::operator<<<std::char_traits<char>>(a1, a2, v7, v3);
  std::ostream::operator<<(a1, a2, refptr__ZSt4endlIcSt11char_traitsIcEERSt13basic_ostreamIT_T0_ES6_, v4);
  return std::operator<<<std::char_traits<char>>(a1, a2, "You did a great job! :D", refptr__ZSt4cout);
}
```

可以看到这里已经是全部回显正确的信息,找到调用它的函数

找到一个关键判断,调用条件根据length判断长度,如果为0则正确,如果为1则失败.

![image-20231219233410952](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312192336019.png)

我们在这个if下断

![image-20231219233515848](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312192336020.png)

运行到这里的时候修改zf,运行

![image-20231219233530177](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312192336021.png)