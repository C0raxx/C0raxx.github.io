---
layout:     post
title:      【NSSCTF逆向】《cpp》
subtitle:   CTF
date:       2023-11-17
author:     Corax
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 【CTF】
    - 【逆向】
    - 【NSSCTF】
---

# cpp

这道题目说实话，是我做到现在很难的题目了，涉及了chacha20，class的逆向，虚函数执行函数。



做了两天才勉强把flag给跑出来，对于class，chacha20仍然有诸多不懂。



不过做难题才可以有成长嘛，先写着吧，针对class，chacha20，虚函数后面再开文章谢谢学学吧。



进入题目之后就是如下的程序

![image-20231118173901357](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311181930886.png)

关键出在这三个里面

![image-20231118173926505](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311181930887.png)

前两天刚刚看过虚函数的表现形式，等于说是在指针变量地址写入RTTI表中自定义函数的头（调试版本），然后跳转到函数头，即第一条指令的地方。

下好断点

![image-20231118174414863](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311181930888.png)

跑起来之后在汇编里面看一下

这条语句进行的就是这条call

![image-20231118174500185](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311181930889.png)

步入并反编译

```
{
  __int64 result; // rax
  char v2[8]; // [rsp+20h] [rbp-118h] BYREF
  __int64 v3; // [rsp+28h] [rbp-110h]
  _DWORD *v4; // [rsp+30h] [rbp-108h]
  __int64 v5; // [rsp+38h] [rbp-100h]
  _DWORD *v6; // [rsp+40h] [rbp-F8h]
  __int64 v7; // [rsp+48h] [rbp-F0h]
  _DWORD *v8; // [rsp+50h] [rbp-E8h]
  __int64 v9; // [rsp+58h] [rbp-E0h]
  _DWORD *v10; // [rsp+60h] [rbp-D8h]
  __int64 v11; // [rsp+68h] [rbp-D0h]
  _DWORD *v12; // [rsp+70h] [rbp-C8h]
  __int64 v13; // [rsp+78h] [rbp-C0h]
  _DWORD *v14; // [rsp+80h] [rbp-B8h]
  __int64 v15; // [rsp+88h] [rbp-B0h]
  _DWORD *v16; // [rsp+90h] [rbp-A8h]
  __int64 v17; // [rsp+98h] [rbp-A0h]
  _DWORD *v18; // [rsp+A0h] [rbp-98h]
  __int64 v19; // [rsp+A8h] [rbp-90h]
  _DWORD *v20; // [rsp+B0h] [rbp-88h]
  __int64 v21; // [rsp+B8h] [rbp-80h]
  _DWORD *v22; // [rsp+C0h] [rbp-78h]
  __int64 v23; // [rsp+C8h] [rbp-70h]
  _DWORD *v24; // [rsp+D0h] [rbp-68h]
  __int64 v25; // [rsp+D8h] [rbp-60h]
  _DWORD *v26; // [rsp+E0h] [rbp-58h]
  __int64 v27; // [rsp+E8h] [rbp-50h]
  _DWORD *v28; // [rsp+F0h] [rbp-48h]
  __int64 v29; // [rsp+F8h] [rbp-40h]
  _DWORD *v30; // [rsp+100h] [rbp-38h]
  __int64 v31; // [rsp+108h] [rbp-30h]
  _DWORD *v32; // [rsp+110h] [rbp-28h]
  __int64 v33; // [rsp+118h] [rbp-20h]
  _DWORD *v34; // [rsp+120h] [rbp-18h]

  sub_7FF7BDD04DF0(a1 + 64, 16i64, v2);
  v3 = a1 + 64;
  v4 = *(_DWORD **)(a1 + 64);
  *v4 = 1634760805;
  v5 = a1 + 64;
  v6 = (_DWORD *)(*(_QWORD *)(a1 + 64) + 4i64);
  *v6 = 857760878;
  v7 = a1 + 64;
  v8 = (_DWORD *)(*(_QWORD *)(a1 + 64) + 8i64);
  *v8 = 2036477234;
  v9 = a1 + 64;
  v10 = (_DWORD *)(*(_QWORD *)(a1 + 64) + 12i64);
  *v10 = 1797285236;
  v11 = a1 + 64;
  v12 = (_DWORD *)(*(_QWORD *)(a1 + 64) + 16i64);
  *v12 = **(unsigned __int8 **)(a1 + 88);
  v13 = a1 + 64;
  v14 = (_DWORD *)(*(_QWORD *)(a1 + 64) + 20i64);
  *v14 = *(unsigned __int8 *)(*(_QWORD *)(a1 + 88) + 1i64);
  v15 = a1 + 64;
  v16 = (_DWORD *)(*(_QWORD *)(a1 + 64) + 24i64);
  *v16 = *(unsigned __int8 *)(*(_QWORD *)(a1 + 88) + 2i64);
  v17 = a1 + 64;
  v18 = (_DWORD *)(*(_QWORD *)(a1 + 64) + 28i64);
  *v18 = *(unsigned __int8 *)(*(_QWORD *)(a1 + 88) + 3i64);
  v19 = a1 + 64;
  v20 = (_DWORD *)(*(_QWORD *)(a1 + 64) + 32i64);
  *v20 = *(unsigned __int8 *)(*(_QWORD *)(a1 + 88) + 4i64);
  v21 = a1 + 64;
  v22 = (_DWORD *)(*(_QWORD *)(a1 + 64) + 36i64);
  *v22 = *(unsigned __int8 *)(*(_QWORD *)(a1 + 88) + 5i64);
  v23 = a1 + 64;
  v24 = (_DWORD *)(*(_QWORD *)(a1 + 64) + 40i64);
  *v24 = *(unsigned __int8 *)(*(_QWORD *)(a1 + 88) + 6i64);
  v25 = a1 + 64;
  v26 = (_DWORD *)(*(_QWORD *)(a1 + 64) + 44i64);
  *v26 = *(unsigned __int8 *)(*(_QWORD *)(a1 + 88) + 7i64);
  v27 = a1 + 64;
  v28 = (_DWORD *)(*(_QWORD *)(a1 + 64) + 48i64);
  *v28 = *(_DWORD *)(a1 + 96);
  v29 = a1 + 64;
  v30 = (_DWORD *)(*(_QWORD *)(a1 + 64) + 52i64);
  *v30 = **(unsigned __int8 **)(a1 + 104);
  v31 = a1 + 64;
  v32 = (_DWORD *)(*(_QWORD *)(a1 + 64) + 56i64);
  *v32 = *(unsigned __int8 *)(*(_QWORD *)(a1 + 104) + 1i64);
  v33 = a1 + 64;
  v34 = (_DWORD *)(*(_QWORD *)(a1 + 64) + 60i64);
  result = *(unsigned __int8 *)(*(_QWORD *)(a1 + 104) + 2i64);
  *v34 = result;
  return result;
}
```

讲真的，看不懂思密达。直接出来吧。



准备步入第二个call

![image-20231118183157778](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311181930890.png)

```
__int64 __fastcall sub_7FF7BDD02E60(__int64 a1)
{
  unsigned __int64 i; // [rsp+20h] [rbp-F8h]
  __int64 v3; // [rsp+40h] [rbp-D8h]
  __int64 v4; // [rsp+70h] [rbp-A8h]
  __int64 v5; // [rsp+78h] [rbp-A0h]
  char v6[24]; // [rsp+88h] [rbp-90h] BYREF
  char v7[24]; // [rsp+A0h] [rbp-78h] BYREF
  char v8[32]; // [rsp+B8h] [rbp-60h] BYREF
  _QWORD v9[3]; // [rsp+D8h] [rbp-40h] BYREF
  _QWORD v10[3]; // [rsp+F0h] [rbp-28h] BYREF

  memset(v10, 0, sizeof(v10));
  (*(void (__fastcall **)(__int64, _QWORD *))(*(_QWORD *)a1 + 24i64))(a1, v10);
  memset(v9, 0, sizeof(v9));
  v3 = sub_7FF7BDD03920(v8, a1 + 8);
  (*(void (__fastcall **)(__int64, _QWORD *, __int64))(*(_QWORD *)a1 + 32i64))(a1, v9, v3);
  for ( i = 0i64; i < (__int64)(v9[1] - v9[0]) >> 2; ++i )
    *(_DWORD *)(v9[0] + 4 * i) ^= *(_DWORD *)(v10[0] + 4 * i);
  v4 = sub_7FF7BDD03580(v7, v9);
  v5 = (*(__int64 (__fastcall **)(__int64, char *, __int64))(*(_QWORD *)a1 + 40i64))(a1, v6, v4);
  sub_7FF7BDD03650(a1 + 40, v5);
  sub_7FF7BDD03DE0(v6);
  sub_7FF7BDD03CA0(v9);
  return sub_7FF7BDD03CA0(v10);
}
```

这个好看很多了，不过先不急。看看第三个



```
__int64 __fastcall sub_7FF7BDD03080(__int64 a1)
{
  char v2[40]; // [rsp+28h] [rbp-90h] BYREF
  char v3; // [rsp+50h] [rbp-68h] BYREF
  char v4[8]; // [rsp+58h] [rbp-60h] BYREF
  __int64 v5[2]; // [rsp+60h] [rbp-58h] BYREF
  char v6[16]; // [rsp+70h] [rbp-48h] BYREF
  char v7[24]; // [rsp+80h] [rbp-38h] BYREF

  memset(v7, 0, sizeof(v7));
  qmemcpy(v2, "(P", 2);
  v2[2] = -63;
  v2[3] = 35;
  v2[4] = -104;
  v2[5] = -95;
  v2[6] = 65;
  v2[7] = 54;
  v2[8] = 76;
  v2[9] = 49;
  v2[10] = -53;
  v2[11] = 82;
  v2[12] = -112;
  v2[13] = -15;
  v2[14] = -84;
  v2[15] = -52;
  v2[16] = 15;
  v2[17] = 108;
  v2[18] = 42;
  v2[19] = -119;
  v2[20] = 127;
  v2[21] = -33;
  v2[22] = 17;
  v2[23] = -124;
  v2[24] = 127;
  v2[25] = -26;
  v2[26] = -94;
  v2[27] = -32;
  v2[28] = 89;
  v2[29] = -57;
  v2[30] = -59;
  v2[31] = 70;
  v2[32] = 93;
  v2[33] = 41;
  v2[34] = 56;
  v2[35] = -109;
  v2[36] = -19;
  v2[37] = 21;
  v2[38] = 122;
  v2[39] = -1;
  v5[0] = (__int64)v2;
  v5[1] = (__int64)&v3;
  qmemcpy(v6, v5, sizeof(v6));
  sub_7FF7BDD03730(v7, v6, v4);
  if ( (unsigned __int8)sub_7FF7BDD04990(v7, a1 + 40) )
  {
    sub_7FF7BDD03DE0(v7);
    return 1i64;
  }
  else
  {
    sub_7FF7BDD03DE0(v7);
    return 0i64;
  }
}
```

第三个根据刚刚在外面的信息

是在if语句里面调用的东西，并且关系到是否正确，函数代码里面也有cpy，可以断定他就是一个判断函数。

然后密文就是这里的v2，看一下他的长短，40位。等会我们用40字长的字符串去试试。

![image-20231118183424345](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311181930891.png)





现在三个函数都看过了，第三个函数的功用大概明白了，现在第一个和第二个还不明确。



第一个看起来像个什么处理，说真的，完全看不出来是处理了什么东西。看内存也看不出来。

看第二个吧。

第二个看起来像一个加密的函数

![image-20231118185941281](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311181930892.png)

主要是看到这里有一个异或的函数。

我们根据地址信息，找到这个地方，就是这个xor，我们跑跑看

![image-20231118190032106](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311181930893.png)

for的第一轮，注意到eax是我们的1234，不过由于端序问题，读进去的时候应该是4321，也就是说我们输入的东西被倒过来了。

![image-20231118190118608](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311181930894.png)

然后rcx是一个序列。

第二轮，rcx的序列产生了变化，rax同样。

![image-20231118190308344](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311181930896.png)

后面十轮都是这样的，试了一下输入不同的字符串，rcx都是一样的，有理由觉得这个东西是key，进行异或加密，十个key如下。

```
key =  [0x4037A04E, 0x0FDDA0246, 0x3C6EFA21, 0x0CF9CD9AF, 0x673347B9, 0x0DEC4EE0, 0x1380C4D1, 0x3AB2A932, 0x25D50A7, 0x834A3982]
```



然后进入第三个函数



在他赋值结束之后再下断点

![image-20231118192740756](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311181930897.png)

获得密文

```
23C15028h, 3641A198h, 52CB314Ch, 0CCACF190h, 892A6C0Fh, 8411DF7Fh, 0E0A2E67Fh, 46C5C759h, 9338295Dh, 0FF7A15EDh
```

要注意的是这个flag是逆序存放的，所以在解的时候还是要反过来

偷的脚本

```
flag = [0x23C15028, 0x3641A198, 0x52CB314C, 0x0CCACF190, 0x892A6C0F, 0x8411DF7F, 0x0E0A2E67F, 0x46C5C759, 0x9338295D, 0xFF7A15ED]

key =  [0x4037A04E, 0x0FDDA0246, 0x3C6EFA21, 0x0CF9CD9AF, 0x673347B9, 0x0DEC4EE0, 0x1380C4D1, 0x3AB2A932, 0x25D50A7, 0x834A3982]

for i in range(0, len(flag)):
    x = bytes.fromhex(hex(flag[i])[2:].rjust(8,'0'))[::-1]
    z = int(x.hex(),16) ^ key[i]
    print(z.to_bytes(4,'big').decode(),end='')
```

hgame{Cpp_1s_much_m0r3_dlff1cult_th4n_C}