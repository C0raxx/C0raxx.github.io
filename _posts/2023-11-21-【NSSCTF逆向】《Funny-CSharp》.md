---
layout:     post
title:      【NSSCTF逆向】《Funny CSharp》
subtitle:   CTF
date:       2023-11-21
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【CTF】
    - 【逆向】
    - 【NSSCTF】
---

# Funny CSharp

 据说是Csharp的题目

![image-20231121142930524](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311221458124.png)

主程序是用VSC写的，那应该是附带的dll。

原本用了一个感觉有问题的dnspy

挺坑的，读不出来函数里面有啥东西

![image-20231121143053586](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311221458126.png)

上52换了一个

ok了

![image-20231121143117052](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311221458127.png)

```
internal static bool <<Main>$>g__IsFlag|0_0(string flag)
	{
		if (flag == null)
		{
			return false;
		}
		byte[] array = Program.<<Main>$>g__ReverseString|0_1(Encoding.UTF8.GetBytes(flag));
		byte[] array2 = new byte[]
		{
			116,
			94,
			80,
			127,
			49,
			71,
			101,
			106,
			115,
			97,
			57,
			83,
			94,
			109,
			58,
			98,
			75,
			86,
			40,
			100,
			61,
			123,
			110,
			57,
			123,
			89,
			86,
			121,
			123,
			104,
			97,
			60,
			86,
			74,
			86,
			112,
			103,
			103,
			124,
			111,
			86,
			99,
			107,
			56,
			93,
			71,
			96,
			106,
			69,
			49,
			112,
			59,
			103,
			124,
			90,
			65,
			71,
			114,
			79,
			93,
			74,
			90,
			90,
			71
		};
		for (int i = 0; i < array.Length; i++)
		{
			byte[] array3 = array;
			int num = i;
			array3[num] ^= 9;
			if (array[i] != array2[i])
			{
				return false;
			}
		}
		return true;
	}

	// Token: 0x06000009 RID: 9 RVA: 0x00001D24 File Offset: 0x00001D24
	[NullableContext(1)]
	[CompilerGenerated]
	internal static byte[] <<Main>$>g__ReverseString|0_1(byte[] p)
	{
		int i = 0;
		int num = p.Length - 1;
		while (i < num)
		{
			ref byte ptr = ref p[i];
			ref byte ptr2 = ref p[num];
			byte b = p[num];
			byte b2 = p[i];
			ptr = b;
			ptr2 = b2;
			i++;
			num--;
		}
		return p;
	}
}
```

程序很简单，把我们的输入逆序，然后异或9，和密文进行比较。

脚本如下：

```
a =[116,
		94,
		80,
		127,
		49,
		71,
		101,
		106,
		115,
		97,
		57,
		83,
		94,
		109,
		58,
		98,
		75,
		86,
		40,
		100,
		61,
		123,
		110,
		57,
		123,
		89,
		86,
		121,
		123,
		104,
		97,
		60,
		86,
		74,
		86,
		112,
		103,
		103,
		124,
		111,
		86,
		99,
		107,
		56,
		93,
		71,
		96,
		106,
		69,
		49,
		112,
		59,
		103,
		124,
		90,
		65,
		71,
		114,
		79,
		93,
		74,
		90,
		90,
		71]
for i in a[::-1]:
    print(chr(i^0x9),end='')
```

![image-20231121143230741](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311221458128.png)

PS：

不明白这里这个异或是怎么生效的，还是反编译错误

![image-20231121143303325](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311221458129.png)

用到了吗？这里的arry3作为变量存在内存里面，和本来的array又不没指针关系。。

