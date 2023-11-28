---
layout:     post
title:      【NSSCTF刷题】《Tea_apk》
subtitle:   CTF
date:       2023-11-28
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【CTF】
    - 【逆向】
    - 【NSSCTF】
---

# 题目 Tea_apk



![image-20231129005604428](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311290117885.png)

## 解法

![image-20231129005629810](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311290117886.png)

下载出来一道apk，直接打开jadx反编译

进去之后搜索MainActivity

就可以找到如下主体部分

![image-20231129005936995](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311290117887.png)

代码逻辑还挺清晰的，调用一个check，传入我们输入的flag

进入check看看

结构如下，调用完 encryptToBase64String(str, "ABvWW7hqwNvHUhfP")的equal方法

也就是encryptToBase64String结果和vlgg9nNjUcYuWzBSSOwKxbMD2rhFgf4zuiyMpLxpNkM=比较，这就是我们的密文

![image-20231129010220343](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311290117888.png)



那我们往上看，分析encryptToBase64String

这玩意不是一个简单的base64

关键的是encrypt

![image-20231129010845288](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311290117890.png)

来到这里

再次调用，不过根据参数类型重用另一个函数



![image-20231129011126189](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311290117891.png)

里面两个是把他调整成合适格式的，不用管

又用了encryp

![image-20231129011155432](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311290117892.png)



然后来到了这个关键代码

```
  private static int[] encrypt(int[] iArr, int[] iArr2) {
        int length = iArr.length - 1;
        if (length < 1) {
            return iArr;
        }
        int i = (52 / (length + 1)) + 6;
        int i2 = iArr[length];
        int i3 = 0;
        while (true) {
            int i4 = i - 1;
            if (i <= 0) {
                return iArr;
            }
            i3 += DELTA;
            i = (i3 >>> 2) & 3;
            int i5 = i2;
            i2 = 0;
            while (i2 < length) {
                int i6 = i2 + 1;
                i5 = iArr[i2] + MX(i3, iArr[i6], i5, i2, i, iArr2);
                iArr[i2] = i5;
                i2 = i6;
            }
            int i7 = iArr[0];
            i2 = MX(i3, i7, i5, i2, i, iArr2) + iArr[length];
            iArr[length] = i2;
            i = i4;
        }
    }

```

i = (52 / (length + 1)) + 6 

这处特征告诉我们这是**xxtea**



现在正向流程明显了

1. xxtea
2. base64加密
3. 比较

逆向也就是

1. 拿到密文
2. base64解密
3. 解xxtea



条件充足，直接上脚本

```
import base64
import xxtea

c =base64.b64decode('vlgg9nNjUcYuWzBSSOwKxbMD2rhFgf4zuiyMpLxpNkM=')
#c = 'vlgg9nNjUcYuWzBSSOwKxbMD2rhFgf4zuiyMpLxpNkM='

#print(c)
key=b'ABvWW7hqwNvHUhfP'
sAns = "cxyyy"
print(xxtea.decrypt(c,key))
bTemp = xxtea.encrypt(sAns,key)
print(base64.b64encode(bTemp))
```

![image-20231129011510781](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311290117893.png)

就是这样









一些疑惑

这道题明显是需要先解base64 再解xxtea的，脚本也正确，但是有一些在线工具确实这样

![image-20231129011614691](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202311290117894.png)



拿待解base64的密文直接解开

我不明白为什么，难道xxtea的结果就一定是不可见字符串，所以它内置了base64吗？