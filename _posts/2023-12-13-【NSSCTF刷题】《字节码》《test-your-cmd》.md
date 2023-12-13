---
layout:     post
title:      【NSSCTF刷题】《字节码》《test your cmd》
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

# 字节码

## 题目

![image-20231214010421578](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312140118830.png)

## 解法

总的来讲虽然题目简单还是有点收获的

出来是一个字节码txt

```
              6 STORE_FAST               1 (s)

  5           8 BUILD_LIST               0
             10 LOAD_CONST               2 ((177, 171, 170, 185, 167, 180, 126, 136, 126, 147, 150, 146, 122, 126, 142, 129, 139, 137, 142, 117, 122, 130, 195, 132, 116, 109, 104))
             12 LIST_EXTEND              1
             14 STORE_FAST               2 (tmp)

  6          16 LOAD_GLOBAL              0 (range)
             18 LOAD_GLOBAL              1 (len)
             20 LOAD_FAST                0 (flag)
             22 CALL_FUNCTION            1
             24 CALL_FUNCTION            1
             26 GET_ITER
        >>   28 FOR_ITER                30 (to 60)
             30 STORE_FAST               3 (i)

  7          32 LOAD_FAST                1 (s)
             34 LOAD_METHOD              2 (append)
             36 LOAD_CONST               3 (255)
             38 LOAD_GLOBAL              3 (ord)
             40 LOAD_FAST                0 (flag)
             42 LOAD_FAST                3 (i)
             44 BINARY_SUBSCR
             46 CALL_FUNCTION            1
             48 LOAD_FAST                3 (i)
             50 BINARY_ADD
             52 BINARY_XOR
             54 CALL_METHOD              1
             56 POP_TOP
             58 JUMP_ABSOLUTE           28

  8     >>   60 LOAD_FAST                2 (tmp)
             62 LOAD_FAST                1 (s)
             64 COMPARE_OP               2 (==)
             66 POP_JUMP_IF_FALSE       78

  9          68 LOAD_GLOBAL              4 (print)
             70 LOAD_CONST               4 ('you are right')
             72 CALL_FUNCTION            1
             74 POP_TOP
             76 JUMP_FORWARD             8 (to 86)

 11     >>   78 LOAD_GLOBAL              4 (print)
             80 LOAD_CONST               5 ('this is wrang')
             82 CALL_FUNCTION            1
             84 POP_TOP
        >>   86 LOAD_CONST               0 (None)
             88 RETURN_VALUE
```

又去详细看了一下字节码的手撕流程

拿这段代码的核心代码举例

```
32 LOAD_FAST                1 (s)
             34 LOAD_METHOD              2 (append)
             36 LOAD_CONST               3 (255)
             38 LOAD_GLOBAL              3 (ord)
             40 LOAD_FAST                0 (flag)
             42 LOAD_FAST                3 (i)
             44 BINARY_SUBSCR
             46 CALL_FUNCTION            1
             48 LOAD_FAST                3 (i)
             50 BINARY_ADD
             52 BINARY_XOR
             54 CALL_METHOD              1
             56 POP_TOP
             58 JUMP_ABSOLUTE           28
```

其实就是栈内元素操作的关系

1. 压入s 我们输入的东西
2. 压入append方法
3. 压入255
4. 压入ord函数
5. 压入flag
6. 压入i变量
7. 然后第一个操作BINARY_SUBSCR 取下标，此时flag和i变成一个栈顶元素flag[i]
8. 然后调用函数CALL_FUNCTION            1其实就是ord这个函数。此时栈顶为ord(flag[i])
9. 压入变量i
10. BINARY_ADD将栈顶ord(flag[i])和i相加(ord(flag[i])+i)
11. BINARY_XOR将栈顶(ord(flag[i])+i)异或255
12. 然后调用方法 append进去

写成python就是(ord(flag[i])+i)^255

其实只要慢慢解就可以了，但是掉进去一个坑



写成这样了

问题就是次序问题 会先255 +i 然后和与tmp[i]异或（其实是加号，图片错了）

![image-20231214011320076](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312140118831.png)

只要加上括号

![image-20231214011515196](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312140118832.png)

或者分开写

![image-20231214011528370](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312140118833.png)

就ok

# test your cmd

## 题目

![image-20231214011617282](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312140118834.png)

## 解法

打开程序果然立马消失了。

ida打开

如图下断点

![image-20231214011652373](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312140118835.png)

因为在结束前下的断点，所以输出内容停住了

![image-20231214011721833](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202312140118836.png)