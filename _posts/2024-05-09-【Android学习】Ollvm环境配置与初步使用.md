---
layout:     post
title:      【Android学习】Ollvm环境配置与初步使用
subtitle:   工具学习
date:       2024-05-09
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【Ollvm】
    - 【逆向】
    - 【Android】
---

# Ollvm环境配置 及 初步使用

```
笔者环境：
虚拟机：系统：   ubuntu22.04
	   分配内存：9G
	   分配空间：800G
	   分配CPU: 8核
	   本身没有任何Cmake，g++版本
```

## 换源

其实本身是挂了梯子的，但是换个源处理可以能更好

```
sudo vim /etc/apt/sources.list
```

开头添加这些源

```
deb http://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse
```

![image-20240510173936146](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202405101803668.png)

## 更新

```
sudo apt update && sudo apt upgrade -y
```

## 安装Cmake

```
sudo apt-get install cmake -y
```

## 安装gcc8 g++8

```
sudo apt-get install gcc-8 g++-8 -y
```

本身虚拟机无版本，无需做将版本处理。

若有版本冲突，可参考文末别的师傅的博文。

安装好之后可以看看版本

![image-20240510174321308](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202405101803669.png)

### 下载LLVM源码

```
git clone -b llvm-4.0  https://github.com/obfuscator-llvm/obfuscator.git
```

如果没git的话自行查阅git下载方法。

## 修改LLVM源码

本身为char，修改为uint8_t

![image-20240510174502974](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202405101803671.png)

## 编译安装LLVM

首先在obfuscator同级目录打开命令行，输入

```
mkdir build
```

效果

![image-20240510174730230](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202405101803672.png)

然后cd进build

```
cd build/
```

### Cmake拉进来

```
cmake -DCMAKE_BUILD_TYPE=Release -DLLVM_INCLUDE_TESTS=OFF ../obfuscator/
```

### make

```
make
```

### sudo make install 

```
sudo make install 
```

这个过程会比较长，而且我在编译过程当中碰到了很多很多警告，最后只要进入bin里面有这四个

![image-20240510175003706](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202405101803673.png)

就能用

## 测试

进入这里的目录

![image-20240510175121120](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202405101803674.png)

```
./clang --version
```

有如下信息应该是没啥问题了

![image-20240510175308162](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202405101803675.png)

## Ollvm初探

装好了我们就来看个效果吧。

自己整了个简单的脚本

调用一个函数 函数里面计算0到15的累加和

```
#include <stdio.h>


int calculateSum();

int main() {
    int sum = calculateSum();
    printf("0 到 15 的累加和为: %d\n", sum);
    return 0;
}

int calculateSum() {
    int sum = 0;
    for (int i = 0; i <= 15; ++i) {
        sum += i;
    }
    return sum;
}

```

我们先最简单编译一下

### 无操作

```
clang test.cpp -o test1
```

main

![image-20240510175826613](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202405101803676.png)

函数

![image-20240510175836928](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202405101803677.png)

眉清目秀

### 尝试控制流平坦化

main因为代码很简单 没啥变化

主要在函数里

![image-20240510175928049](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202405101803678.png)

### 尝试指令替代

![image-20240510180015186](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202405101803679.png)

没啥大变化，但是如果仔细观察的话，还是可以看到多了一些xor,sub指令

### 尝试虚假控制流

![image-20240510180107120](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202405101803680.png)

这块我最近在研究，是标准的bcf样式。

读的vae师傅的虚假控制流源码解读，后面也会写一篇自己的学习心得。

































































