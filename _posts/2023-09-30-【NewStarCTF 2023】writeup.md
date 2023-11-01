---
layout:     post
title:      【NewStarCTF2023】writeup
subtitle:   部分逆向
date:       2023-09-30
author:     Corax
header-img: img/post-bg-digital-native.jpg
catalog: true
tags:
    - 【CTF】
    - 【逆向】
---

## easy_RE

先exeinfo

![image-20230930211120683](https://cdn.jsdelivr.net/gh/C0raxx/blogimage/202309302117649.png)

没有壳，直接上ida

![image-20230930211138278](https://cdn.jsdelivr.net/gh/C0raxx/blogimage/202309302117650.png)

看到有关键信息，但没有显示完，按f5反编译一下

![image-20230930211145993](https://cdn.jsdelivr.net/gh/C0raxx/blogimage/202309302117652.png)

拼接一下输入，注意字符串的部分字母由近似数字代替，提交

![image-20230930211155158](https://cdn.jsdelivr.net/gh/C0raxx/blogimage/202309302117653.png)

## KE

![image-20230930211204010](https://cdn.jsdelivr.net/gh/C0raxx/blogimage/202309302117654.png)

运行一下，看到是KE，想着可能是壳的意思？exeinfo看一下

![image-20230930211212039](https://cdn.jsdelivr.net/gh/C0raxx/blogimage/202309302117655.png)

upx壳，尝试upx脱壳

![image-20230930211220876](https://cdn.jsdelivr.net/gh/C0raxx/blogimage/202309302117656.png)

呃呃 权限好像不对 尝试了一下用管理员权限打开

![image-20230930211228359](https://cdn.jsdelivr.net/gh/C0raxx/blogimage/202309302117657.png)

要记住先输入红框中的两行指令才可以打开目标文件夹。

打开之后可以看到成功脱壳了。

直接ida f5看看

![image-20230930211236218](https://cdn.jsdelivr.net/gh/C0raxx/blogimage/202309302117658.png)

程序逻辑也很清晰了，第一个红框输入字符并且通过++Str1[i]把字符的ascii值往后推一个，第二个红框就是加密之后与字符串enc进行比较

enc：

![image-20230930211244473](https://cdn.jsdelivr.net/gh/C0raxx/blogimage/202309302117659.png)

写脚本吧

```
a="gmbh|D1ohsbuv2bu21ot1oQb332ohUifG2stuQ[HBMBYZ2fwf2~"
for i in a:
    k=ord(i)-1
    print(chr(k),end='')
```

出flag

`flag{C0ngratu1at10ns0nPa221ngTheF1rstPZGALAXY1eve1}`

## segment

![image-20230930211351023](https://cdn.jsdelivr.net/gh/C0raxx/blogimage/202309302117660.png)

elf，那放到kali里面看看

![image-20230930211358249](https://cdn.jsdelivr.net/gh/C0raxx/blogimage/202309302117661.png)

额，没啥信息，放ida里面吧，因为题目的信息也是这个

![image-20230930211410184](https://cdn.jsdelivr.net/gh/C0raxx/blogimage/202309302117662.png)

ok，啥也没有，按照提示shift f7打开段窗口

![image-20230930211417620](https://cdn.jsdelivr.net/gh/C0raxx/blogimage/202309302117663.png)

关键信息出来了，但是要注意的是flag后面的和name后面的__要换成{}

`flag{You_ar3_g0od_at_f1nding_ELF_segments_name}`

## ELF

![image-20230930211431820](https://cdn.jsdelivr.net/gh/C0raxx/blogimage/202309302117664.png)

感觉没啥特殊的，扔kali里面看看

![image-20230930211438635](https://cdn.jsdelivr.net/gh/C0raxx/blogimage/202309302117665.png)

ida里面看看

![image-20230930211447087](https://cdn.jsdelivr.net/gh/C0raxx/blogimage/202309302117666.png)

可以看到，我们输入的字符串先经过了两重加密，然后和Vlx…进行比较。

所以你想思路就是Vlx…先解base64，再解上面那个encode。

本来第一步解base64我用的就是在线网站

![image-20230930211454699](https://cdn.jsdelivr.net/gh/C0raxx/blogimage/202309302117667.png)

然后换了一个网站，发现还没什么，直接就进行下一步

![image-20230930211502686](https://cdn.jsdelivr.net/gh/C0raxx/blogimage/202309302117668.png)

上面那个encode的逻辑就是先异或0x20,再减去16

那直接写就是了

`flag{D04ou7nowwha7ELF1s?}`

提交，不对。

后面我就觉得应该是上面有些不是标准字符，但是我错误的把他直接用来解密，就研究怎么直接在python里面，利用base64库，不用显示转义字符地得到字符串。直接告诉gpt需求，就写了

```
import base64

# 密文字符串
encrypted_string = "VlxRV2t0II8kX2WPJ15fZ49nWFEnj3V8do8hYy9t"
# Base64 替代字符集
altchars = b'@~'

# Base64 解码并使用替代字符集解析非标准字符
decoded_bytes = base64.b64decode(encrypted_string, altchars=altchars)

# 使用异或解密
decrypted_bytes = bytes([(c - 16) ^ 0x20 for c in decoded_bytes])

# 得到最初的字符串
original_string = decrypted_bytes.decode('utf-8')

print(original_string)

```

有了

`flag{D0_4ou_7now_wha7_ELF_1s?}`
