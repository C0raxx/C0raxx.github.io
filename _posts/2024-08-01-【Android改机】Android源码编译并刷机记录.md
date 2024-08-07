---
layout:     post
title:      【Android改机】Android源码编译并刷机记录
subtitle:   Android改机
date:       2024-08-01
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【Android改机】
    - 【逆向】
    - 【Android】
---

# Android源码编译并刷机记录

仅以此文用来记录安卓源码编译成功时的一些经验和心得，并将成功编译的 android-10.0.0_r17,pixel3的userdebug版本置于GitHub。

后续会**有关于源码魔改角度出发的Frida持久化，整体壳和抽取壳脱壳相关的内容**，敬请期待。

资源省流

```
编译成果：https://github.com/C0raxx/AndroidCompile/blob/main/Android10/Android10Original/DownLoadUrl
编译环境（初始版，仍需配置）：https://github.com/C0raxx/AndroidCompile/blob/main/Android10/CompileEnv/DownLoadUrl
```

没有配上很多图，因为成果测试的时候没截下来，但可以确定地是按照下面的来，基本上都是**可以成功编译**的。

## 1.虚拟机环境配置

```
VMware版本：17Pro
虚拟机版本：Linux ubuntu 5.15.0-117-generic #127~20.04.1-Ubuntu SMP Thu Jul 11 15:36:12 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
虚拟机硬盘：1T
虚拟机内存：16G
虚拟机处理器：8个
```

因为我整个教程是跟着小肩膀师傅的课程做的，初始环境（无源码无as无clion）也是用的他给的20.04。

```
网络配置
桥接模式 使用真实网卡，可以与主机和外网通讯，有独立ip地址
NAT模式 使用VMnet8网卡，可以与主机和外网通讯，不占用ip地址
Host-only 使用VMnet1网卡，只能与主机通信

我们一般情况下用桥接模式了，因为很多时候用的清华源，宿主机开着代理，我这边进行编译的时候就会被墙。桥接模式下我只要更改主机的代理设置即可，更加方便。
```

## 2.初始化包的下载与解压

```
mkdir ~/bin
cd ~/bin
wget https://mirrors.ustc.edu.cn/aosp-monthly/aosp-latest.tar
md5sum aosp-latest.tar
tar xvf aosp-latest.tar
```

这里注意下载好之后记得拿md5sum算一下值做校验，访问https://mirrors.ustc.edu.cn/aosp-monthly/aosp-latest.tar.md5即可得到你下载的这个初始化包的md5。

这个初始化包更新频率还是挺高的，但是访问我给的url就可以得到最新包的md5。

## 3.配置git

```
sudo apt-get install git
git config --global user.email xxxxxxx@qq.com
git config --global user.name "corax"
```

这一步下载好git，并配置了邮箱。主要是后面进行repo的时候需要这个邮箱。

## 4. 下载repo

```
echo "PATH=~/bin:\$PATH" >> ~/.bashrc
source ~/.bashrc
sudo apt-get install curl
curl -sSL  'https://gerrit-googlesource.proxy.ustclug.org/git-repo/+/master/repo?format=TEXT' |base64 -d > ~/bin/repo
chmod a+x ~/bin/repo
export REPO_URL='https://gerrit-googlesource.proxy.ustclug.org/git-repo'
cd aosp
```

这也没啥好讲的，一样操作就行，我没遇到什么报错。

## 5.修改默认Python

ubuntu内置的python是python2下的某个版本，编译的话我们选择3.8，是已知能成功的。我这之前下过3.8了，没下过的下一下，也挺方便。

然后

```
sudo unlink /usr/bin/python
sudo ln -s /usr/bin/python3.8 /usr/bin/python
```

后面我们执行python就是python3.8进行解释编译了。

## 6.同步指定版本源码

先有一个问题，我们需要啥版本的源码呢？我们参考下面的url

https://source.android.com/docs/setup/about/build-numbers?hl=zh_cn

比方说我们要编译源码之后放在pixel3上面运行，那我们就搜pixel 3，注意不能是pxiel3xl等等。

![image-20240807153225361](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712803.png)

我们需要安卓10的，就往下，尽量找个适配版本多的，后续不用再同步了。

就用这个

![image-20240807153353510](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712804.png)

现在选好版本了，但是还不能做同步的初始化，因为repo需要以下设置，这个我也踩过坑，这里直接避过去。

```
cd ~/bin/aosp/.repo/repo
git pull
cd ~/bin/aosp
```

然后再

```
repo init -u git://mirrors.ustc.edu.cn/aosp/platform/manifest -b android-10.0.0_r17
repo sync
```

这里也会比较长时间，耐心等待吧。

## 7.安装JDK8

其实到这里，源码下载就ok了，到源码编译的准备了。

首先重要的就是JDK的安装，安卓10用JDK8就ok

```
sudo add-apt-repository ppa:openjdk-r/ppa
sudo apt-get update
sudo apt-get install openjdk-8-jdk
```

## 8.安装依赖

```
sudo apt-get install git-core gnupg flex bison build-essential zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip fontconfig libncurses5
```

## 9.设备驱动准备

进行到这里我们下载的源码里面缺少一个文件就是，vendor。

我们编译出一个安卓系统之后还需要在手机上运行，而手机上的硬件需要驱动来进行使用，这一步就是咱们把驱动进程进源码环境，让它后面编译的时候直接用。

https://developers.google.com/android/drivers

我们直接用上面的编号搜

![image-20240807154341905](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712805.png)

找到这个

![image-20240807154402077](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712806.png)

两个下载下来

然后释放进aosp，也就是如下目录

![image-20240807154513777](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712807.png)

解压出来两个sh，运行它

```
./extract-google_devices-blueline.sh
./extract-qcom-blueline.sh
```

ok

## 10.编译源码

```
source build/envsetup.sh
lunch
//这里选择要进行编译的版本
//关于设备代号百度一下就可以 比如pixel3就叫blueline
//然后选择编译版本 userdebug eng user
//userdebug 调试版本 我们用这个
//eng 工程版本
//user 发行版本
//可能编译的时候看不到 除了userdebug的其他选项，这个需要进行设置，但我们要userdebug，暂且不管
make -j8
```

编译过程中会碰到一些警告这都没事，只要不停都是ok的。

如果你不是改变需要运行的设备，比如从pixel3到pixel5，那都不需要make clean。改了源码直接就make上了。

若途中意外退出，继续make即可。

## 11.刷机包下载

```
刷机方式的分类
线刷 官方包(刷的比较彻底，可以刷bootloader、radio)
卡刷 lineage os(双清)

线刷包/工厂镜像包
OTA全量包/OTA增量包
```

我们讨巧一点，直接用原版的刷机包

https://developers.google.com/android/images

用编号搜

![image-20240807155454970](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712808.png)

![image-20240807155459839](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712809.png)

下载下来解压

## 12.刷机包替换

咱们就直接打开里面的这个

![image-20240807155547917](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712810.png)

然后看里面有啥东西，直接从编译出来的东西里面拖出来，复制进去（这里不要解压缩再添加压缩文件）

![image-20240807155637085](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712811.png)

在这，拖出来，然后复制进去。看到时间变新即可。（我这已经拖过了）

![image-20240807155658837](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712812.png)

## 13.刷机

直接傻瓜式

先脸上手机

```
adb reboot bootloader
```

进入bootloader然后点下面这个bat，精心等待即可。

![image-20240807155732461](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712813.png)