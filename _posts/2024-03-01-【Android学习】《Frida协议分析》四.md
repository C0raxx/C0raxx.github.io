---
layout:     post
title:      【Android学习】《Frida协议分析》四
subtitle:   工具学习
date:       2024-03-01
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【Android应用安全实战：Frida协议分析】
    - 【逆向】
    - 【Android】
---

## 第四章

利用Frida框架可以构建出许多强大而高效的工具，本章介绍的“自吐”算法框架本质上只是对一些系统加密函数进行了Hook, Objection也只是对Frida框架进行了更高层的封装，但是却极大提升了逆向工程的效率。由此可见，使用Frida框架封装构建更顺手的工具是一件值得花费时间去实践的事情。



算法“自吐”脚本开发

```
只要把现行常用的密码学加密的通用方法进行Hook，就可以覆盖市面上大部分的Android应用了，配合堆栈打印后还能直接定位到加密点
```

###  工具函数封装

是先编写算法框架中的一些常用函数，如对传入的参数和输出的结果的编码，关键加密函数的定位等。

第一个封装的是堆栈打印，即showStack

封装的是getStackTraceString

```
Java.perform(function(){
    function showStack(){
        var stack=Java.use("android.util.Log").getStackTraceString(
            Java.use("java.lang.Throwable").$new());
            console.log(stack);
    };
})
```

接着封装Base64编码函数。在Hook时，通常需要打印传入参数和输出的加密结果，这就需要进行编码显示，因为开发者不可能直接看懂晦涩的Byte数组，可以将其转化为UTF-8、Hex或者Base64形式进行显示。此外，进行算法Hook时，还需要进行网络抓包，根据抓包结果搜索想要的关键字。

分为两种方法 一个是直接用JavaScript写，另外是java层现成函数调用

```
    function toBase64(data){
        var ByteString=Java.use("com.android.okhttp.okio.ByteString");
        console.log("ByteString:",ByteString);
        console.log(ByteString.of(data).base64());
    }
```

UTF-8编码和Hex编码 一样的原理

```
Java.perform(function(){
    var ByteString = Java.use("com.android.okhttp.okio.ByteString");
    function toBase64(data) {
        console.log(" Base64: ", ByteString.of(data).base64());
    }
    function toHex(data) {
        console.log(" Hex: ", ByteString.of(data).hex());
    }
    function toUtf8(data) {
        console.log(" Utf8: ", ByteString.of(data).utf8());
    }
    toBase64([48,49,50,51,52]);
    toUtf8([48,49,50,51,52]);
    toHex([48,49,50,51,52]);
})
```

### Frida Hook MD5算法

```
目前，HASH函数主要有MDx系列和SHA系列，MDx系列包括MD5、HAVAL、RIPEMD-128等。SHA系列包括SHA-0、SHA-1、SHA-256等。在HASH算法中，MD5和SHA1是应用最广泛的，两者原理差不多，但MD5加密后为128位，SHA加密后为160位。HASH函数也称为杂凑函数或杂凑算法，它是一种把任意长度的输入消息串变化成固定长度的输出串的函数，这个输出串称为该消息的HASH值。也可以说，HASH函数用于找到一种数据内容和数据存放地址之间的映射关系，由于输入值大于输出值，因此不同的输入一定有相同的输出，但因为空间非常大，很难找出，所以可以把HASH函数值看成伪随机数。
```

java写md5一般是使用java.security.MessageDigest，初始化后使用update方法进行处理，一旦准备更新的数据所有都被更新就调用digest完成哈希计算，要hook其中的update和digest方法

测试用例如下

![image-20240307140256550](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081407522.png)

#### Hook MD5算法update方法

```
    var messageDigest = Java.use("java.security.MessageDigest");
    messageDigest.update.implementation = function (data) {}

 var messageDigest = Java.use("java.security.MessageDigest");
    messageDigest.update.overload('byte').implementation = function (data) {
        console.log("MessageDigest.update('byte') is called!");
        return this.update(data);
    }
    messageDigest.update.overload('java.nio.ByteBuffer').implementation = function (data) {
        console.log("MessageDigest.update('java.nio.ByteBuffer') is called!");
        return this.update(data);
    }
    messageDigest.update.overload('[B').implementation = function (data) {
        console.log("MessageDigest.update('[B') is called!");
        var algorithm = this.getAlgorithm();
        var tag = algorithm + " update data";
        toUtf8(tag, data);
        toHex(tag, data);
        toBase64(tag, data);
        console.log("=================================================");
        return this.update(data);
    }
    messageDigest.update.overload('[B', 'int', 'int').implementation = 
function (data, start, length) {
        console.log("MessageDigest.update('[B', 'int', 'int') is called!");
        var algorithm = this.getAlgorithm();
        var tag = algorithm + " update data";
        toUtf8(tag, data);
        toHex(tag, data);
        toBase64(tag, data);
        console.log("=========================================", start, length);
        return this.update(data, start, length);
    }
```

第一步，对每个重载方法的Hook，先进行console打印，提示该方法被调用。由于前两个方法不太常用，因此只进行console打印，如果Android应用确实使用了该方法，可以通过堆栈打印找到加密点进行查看。后两种方法比较常用，因此对其中的参数进行编码转化，此外，使用getAlgorithm方法返回一个标识算法的字符串，可以通过它得到方法名，因为这个方法是一个非静态方法，所以通过this进行调用。

第二步，在编码转化中添加tag参数，标注调用的方法名，方便日志查看。update方法用于更新摘要，也不存在返回值的输出，不需要进行结果的编码转化，所以最后直接调用原方法返回即可。

#### Hook MD5算法digest方法

多个重载

```
 messageDigest.digest.implementation = function () {}

  messageDigest.digest.overload().implementation = function () {
        console.log("MessageDigest.digest() is called!");
        var result = this.digest();
        var algorithm = this.getAlgorithm();
        var tag = algorithm + " digest result";
        toHex(tag, result);
        toBase64(tag, result);
       console.log("================================");
        return result;
    }

 messageDigest.digest.overload('[B').implementation = function (data) {
        console.log("MessageDigest.digest('[B') is called!");
        var algorithm = this.getAlgorithm();
        var tag = algorithm + " digest data";
        toUtf8(tag, data);
        toHex(tag, data);
        toBase64(tag, data);
        var result = this.digest(data);
        var tags = algorithm + " digest result";
        toHex(tags, result);
        toBase64(tags, result);
        console.log(("================================");
        return result;
    }

  messageDigest.digest.overload('[B', 'int', 'int').implementation = 
function (data, start, length) {
        console.log("MessageDigest.digest('[B', 'int', 'int') is called!");
        var algorithm = this.getAlgorithm();
        var tag = algorithm + " digest data";
        toUtf8(tag, data);
        toHex(tag, data);
        toBase64(tag, data);
        var result = this.digest(data, start, length);
        var tags = algorithm + " digest result";
        toHex(tags, result);
        toBase64(tags, result);
        console.log("============================", start, length);
        return result;
    }
```

### Frida Hook MAC算法

首先会讲解何为MAC算法，接着会分别Hook MAC算法中关键的update方法和doFinal方法。

```
MAC算法即消息认证码算法，作为一种可携带密钥的hash函数，通常用来检验所传输消息的完整性。该算法综合了MD和SHA算法的特性，和MD、SHA算法类似，但在此基础上加上了密钥。在HTTP中使用最多的MAC算法是HMAC算法。
```

主要是算法和key

```
该算法通过SecreKeySpec生成密钥，其中的密钥来自于javax.crypto.spec.SecretKeySpec类，它可以用于从字节数组构造一个SecretKey，之后用静态方法getInstance返回实现指定MAC算法的Mac对象，再使用生成的密钥来初始化Mac对象，最后通过update和doFinal方法完成Mac操作。要得到密钥，可以选择Hook SecretKeySpec对象或者init方法，此外，update方法和doFinal方法中的参数也需要获取。
```

源码如下

![image-20240307140541153](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081407523.png)

#### Hook MAC算法密钥

两种思路 Hook SecretKeySpec对象或者init方法

这里对init进行hook 存在重载

```
    var mac = Java.use("javax.crypto.Mac");mac.init.implementation = function () {}

 var mac = Java.use("javax.crypto.Mac");
    mac.init.overload('java.security.Key', 
'java.security.spec.AlgorithmParameterSpec').implementation = function (key, 
AlgorithmParameterSpec) {
        console.log("Mac.init('java.security.Key', 
'java.security.spec.AlgorithmParameterSpec') is called!");
        return this.init(key, AlgorithmParameterSpec);
    }
    mac.init.overload('java.security.Key').implementation = function (key) {
        console.log("Mac.init('java.security.Key') is called!");
        var algorithm = this.getAlgorithm();
        var tag = algorithm + " init Key";
        var keyBytes = key.getEncoded();
        toUtf8(tag, keyBytes);
        toHex(tag, keyBytes);
        toBase64(tag, keyBytes);
        console.log("=======================================");
        return this.init(key);
    }
```

#### Hook MAC算法update方法

它也有update 和md5 只需要对类名和内部修改一下就可以

```
 mac.update.overload('byte').implementation = function (data) {
        console.log("Mac.update('byte') is called!");
        return this.update(data);
    }
    mac.update.overload('java.nio.ByteBuffer').implementation = function (data) {
        console.log("Mac.update('java.nio.ByteBuffer') is called!");
        return this.update(data);
    }
    mac.update.overload('[B').implementation = function (data) {
        console.log("Mac.update('[B') is called!");
        var algorithm = this.getAlgorithm();
        var tag = algorithm + " update data";
        toUtf8(tag, data);
        toHex(tag, data);
        toBase64(tag, data);
        console.log("======================================");
        return this.update(data);
    }
    mac.update.overload('[B', 'int', 'int').implementation = function (data, start, length) {
        console.log("Mac.update('[B', 'int', 'int') is called!");
        var algorithm = this.getAlgorithm();
        var tag = algorithm + " update data";
        toUtf8(tag, data);
        toHex(tag, data);
        toBase64(tag, data);
        console.log("==================================", start, length);
        return this.update(data, start, length);
    }
```

，会发现update方法输出了两次。原来doFinal方法内部还是会调用update方法进行处理，因此接下来Hook MAC算法的doFinal方法时，就不需要对输入参数进行打印输出了，只需要关注输出结果即可

#### Hook MAC算法doFinal方法

与md5 基本一致

```
mac.doFinal.overload().implementation = function () {
        console.log("Mac.doFinal() is called!");
        var result = this.doFinal();
        var algorithm = this.getAlgorithm();
        var tag = algorithm + " doFinal result";
        toHex(tag, result);
        toBase64(tag, result);
        console.log("=======================================");
        return result;
    }
```

### Frida Hook数字签名算法

数字签名的第一步是产生一个需签名的数据的哈希值，第二步是把这个哈希值用私钥加密。

源码

![image-20240307141052031](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081407524.png)

#### Hook数字签名算法update方法

存在四个重载

![image-20240307141151248](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081407525.png)

hook 定位到java.security.Signature类

```
   var signature = Java.use("java.security.Signature");
    signature.update.overload('byte').implementation = function (data) {
        console.log("Signature.update('byte') is called!");
        return this.update(data);
    }
    signature.update.overload('java.nio.ByteBuffer').implementation = function (data) {
        console.log("Signature.update('java.nio.ByteBuffer') is called!");
        return this.update(data);
    }
    signature.update.overload('[B', 'int', 'int').implementation = function (data, start, length) 
{
        console.log("Signature.update('[B', 'int', 'int') is called!");
        var algorithm = this.getAlgorithm();
        var tag = algorithm + " update data";
        toUtf8(tag, data);
        toHex(tag, data);
        toBase64(tag, data);
        console.log("=================================", start, length);
        return this.update(data, start, length);
    }
```

会发现update方法输出了4次，其实前两次是客户端的update方法，后两次是验证的update方法，又因为只含有一个参数的重载方法底层调用了有三个参数的重载方法，因此查看输出结果时，只需要查看拥有三个参数的重载方法的输出即可。

#### Hook数字签名算法sign方法

两个重载

![image-20240307141653983](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081407526.png)

对无参数进行hook

```
 signature.sign.overload('[B', 'int', 'int').implementation = function () {
        console.log("Signature.sign('[B', 'int', 'int') is called!");
        return this.sign.apply(this, arguments);
    }
 signature.sign.overload().implementation = function () {
        console.log("Signature.sign() is called!");
        var result = this.sign();
        var algorithm = this.getAlgorithm();
        var tag = algorithm + " sign result";
        toHex(tag, result);
        toBase64(tag, result);
        console.log("===================================");
        return result;
    }
```



自吐完整代码

```
Java.perform(function () {
    function showStacks() {
        console.log(
            Java.use("android.util.Log")
                .getStackTraceString(
                    Java.use("java.lang.Throwable").$new()
                )
        );
    }
    var ByteString = Java.use("com.android.okhttp.okio.ByteString");
    function toBase64(tag, data) {
        console.log(tag + " Base64: ", ByteString.of(data).base64());
    }
    function toHex(tag, data) {
        console.log(tag + " Hex: ", ByteString.of(data).hex());
    }
    function toUtf8(tag, data) {
        console.log(tag + " Utf8: ", ByteString.of(data).utf8());
    }
    // toBase64([48,49,50,51,52]);
    // toHex([48,49,50,51,52]);
    // toUtf8([48,49,50,51,52]);
    //console.log(Java.enumerateLoadedClassesSync().join("\n"));

    var messageDigest = Java.use("java.security.MessageDigest");
    messageDigest.update.overload('byte').implementation = function (data) {
        console.log("MessageDigest.update('byte') is called!");
        return this.update(data);
    }
    messageDigest.update.overload('java.nio.ByteBuffer').implementation = function (data) 
{
        console.log("MessageDigest.update('java.nio.ByteBuffer') is called!");
        return this.update(data);
    }
    messageDigest.update.overload('[B').implementation = function (data) {
        console.log("MessageDigest.update('[B') is called!");
        var algorithm = this.getAlgorithm();
        var tag = algorithm + " update data";
        toUtf8(tag, data);
        toHex(tag, data);
        toBase64(tag, data);
        console.log("=================================");
        return this.update(data);
    }
    messageDigest.update.overload('[B', 'int', 'int').implementation = function (data, start, 
length) {
        console.log("MessageDigest.update('[B', 'int', 'int') is called!");
        var algorithm = this.getAlgorithm();
        var tag = algorithm + " update data";
        toUtf8(tag, data);
        toHex(tag, data);
        toBase64(tag, data);
        console.log("==================================", start, length);
        return this.update(data, start, length);
    }
    messageDigest.digest.overload().implementation = function () {
        console.log("MessageDigest.digest() is called!");
        var result = this.digest();
        var algorithm = this.getAlgorithm();
        var tag = algorithm + " digest result";
        toHex(tag, result);
        toBase64(tag, result);
        console.log("==========================");
        return result;
    }
    messageDigest.digest.overload('[B').implementation = function (data) {
        console.log("MessageDigest.digest('[B') is called!");
        var algorithm = this.getAlgorithm();
        var tag = algorithm + " digest data";
        toUtf8(tag, data);
        toHex(tag, data);
        toBase64(tag, data);
        var result = this.digest(data);
        var tags = algorithm + " digest result";
        toHex(tags, result);
        toBase64(tags, result);
        console.log("===========================");
        return result;
    }
    messageDigest.digest.overload('[B', 'int', 'int').implementation = function (data, start, 
length) {
        console.log("MessageDigest.digest('[B', 'int', 'int') is called!");
        var algorithm = this.getAlgorithm();
        var tag = algorithm + " digest data";
        toUtf8(tag, data);
        toHex(tag, data);
        toBase64(tag, data);
        var result = this.digest(data, start, length);
        var tags = algorithm + " digest result";
        toHex(tags, result);
        toBase64(tags, result);
        console.log("============================", start, length);
        return result;
    }

    var mac = Java.use("javax.crypto.Mac");
    mac.init.overload('java.security.Key', 
'java.security.spec.AlgorithmParameterSpec').implementation = function (key, 
AlgorithmParameterSpec) {
        console.log("Mac.init('java.security.Key', 
'java.security.spec.AlgorithmParameterSpec') is called!");
        return this.init(key, AlgorithmParameterSpec);
    }
    mac.init.overload('java.security.Key').implementation = function (key) {
        console.log("Mac.init('java.security.Key') is called!");
        var algorithm = this.getAlgorithm();
        var tag = algorithm + " init Key";
        var keyBytes = key.getEncoded();
        toUtf8(tag, keyBytes);
        toHex(tag, keyBytes);
        toBase64(tag, keyBytes);
        console.log("=================================");
        return this.init(key);
    }
    mac.update.overload('byte').implementation = function (data) {
        console.log("Mac.update('byte') is called!");
        return this.update(data);
    }
    mac.update.overload('java.nio.ByteBuffer').implementation = function (data) {
        console.log("Mac.update('java.nio.ByteBuffer') is called!");
        return this.update(data);
    }
    mac.update.overload('[B').implementation = function (data) {
        console.log("Mac.update('[B') is called!");
        var algorithm = this.getAlgorithm();
        var tag = algorithm + " update data";
        toUtf8(tag, data);
        toHex(tag, data);
        toBase64(tag, data);
        console.log("=============================");
        return this.update(data);
    }
    mac.update.overload('[B', 'int', 'int').implementation = function (data, start, length) {
        console.log("Mac.update('[B', 'int', 'int') is called!");
        var algorithm = this.getAlgorithm();
        var tag = algorithm + " update data";
        toUtf8(tag, data);
        toHex(tag, data);
        toBase64(tag, data);
        console.log("===========================", start, length);
        return this.update(data, start, length);
    }
    mac.doFinal.overload().implementation = function () {
        console.log("Mac.doFinal() is called!");
        var result = this.doFinal();
        var algorithm = this.getAlgorithm();
        var tag = algorithm + " doFinal result";
        toHex(tag, result);
        toBase64(tag, result);
        console.log("=================================");
        return result;
    }

    var cipher = Java.use("javax.crypto.Cipher");
    cipher.init.overload('int', 'java.security.cert.Certificate').implementation = function () {
        console.log("Cipher.init('int', 'java.security.cert.Certificate') is called!");
        return this.init.apply(this, arguments);
    }
    cipher.init.overload('int', 'java.security.Key', 
'java.security.SecureRandom').implementation = function () {
        console.log("Cipher.init('int', 'java.security.Key', 'java.security.SecureRandom') is 
called!");
        return this.init.apply(this, arguments);
    }
    cipher.init.overload('int', 'java.security.cert.Certificate', 
'java.security.SecureRandom').implementation = function () {
        console.log("Cipher.init('int', 'java.security.cert.Certificate', 
'java.security.SecureRandom') is called!");
        return this.init.apply(this, arguments);
    }
    cipher.init.overload('int', 'java.security.Key', 'java.security.AlgorithmParameters', 
'java.security.SecureRandom').implementation = function () {
        console.log("Cipher.init('int', 'java.security.Key', 
'java.security.AlgorithmParameters', 'java.security.SecureRandom') is called!");
        return this.init.apply(this, arguments);
    }
    cipher.init.overload('int', 'java.security.Key', 
'java.security.spec.AlgorithmParameterSpec', 'java.security.SecureRandom').implementation 
= function () {
        console.log("Cipher.init('int', 'java.security.Key', 
'java.security.spec.AlgorithmParameterSpec', 'java.security.SecureRandom') is called!");
        return this.init.apply(this, arguments);
    }
    cipher.init.overload('int', 'java.security.Key', 
'java.security.AlgorithmParameters').implementation = function () {
        console.log("Cipher.init('int', 'java.security.Key', 
'java.security.AlgorithmParameters') is called!");
        return this.init.apply(this, arguments);
    }

    cipher.init.overload('int', 'java.security.Key').implementation = function () {
        console.log("Cipher.init('int', 'java.security.Key') is called!");
        var algorithm = this.getAlgorithm();
        var tag = algorithm + " init Key";
        var className = JSON.stringify(arguments[1]);
        if(className.indexOf("OpenSSLRSAPrivateKey") === -1){
            var keyBytes = arguments[1].getEncoded();
            toUtf8(tag, keyBytes);
            toHex(tag, keyBytes);
            toBase64(tag, keyBytes);
        }
        console.log("=============================");
        return this.init.apply(this, arguments);
    }
    cipher.init.overload('int', 'java.security.Key', 
'java.security.spec.AlgorithmParameterSpec').implementation = function () {
        console.log("Cipher.init('int', 'java.security.Key', 
'java.security.spec.AlgorithmParameterSpec') is called!");
        var algorithm = this.getAlgorithm();
        var tag = algorithm + " init Key";
        var keyBytes = arguments[1].getEncoded();
        toUtf8(tag, keyBytes);
        toHex(tag, keyBytes);
        toBase64(tag, keyBytes);
        var tags = algorithm + " init iv";
        var iv = Java.cast(arguments[2], Java.use("javax.crypto.spec.IvParameterSpec"));
        var ivBytes = iv.getIV();
        toUtf8(tags, ivBytes);
        toHex(tags, ivBytes);
        toBase64(tags, ivBytes);
        console.log("==============================");
        return this.init.apply(this, arguments);
    }

    cipher.doFinal.overload('java.nio.ByteBuffer', 'java.nio.ByteBuffer').implementation = 
function () {
        console.log("Cipher.doFinal('java.nio.ByteBuffer', 'java.nio.ByteBuffer') is 
called!");
        return this.doFinal.apply(this, arguments);
    }
    cipher.doFinal.overload('[B', 'int').implementation = function () {
        console.log("Cipher.doFinal('[B', 'int') is called!");
        return this.doFinal.apply(this, arguments);
    }
    cipher.doFinal.overload('[B', 'int', 'int', '[B').implementation = function () {
        console.log("Cipher.doFinal('[B', 'int', 'int', '[B') is called!");
        return this.doFinal.apply(this, arguments);
    }
    cipher.doFinal.overload('[B', 'int', 'int', '[B', 'int').implementation = function () {
        console.log("Cipher.doFinal('[B', 'int', 'int', '[B', 'int') is called!");
        return this.doFinal.apply(this, arguments);
    }
    cipher.doFinal.overload().implementation = function () {
        console.log("Cipher.doFinal() is called!");
        return this.doFinal.apply(this, arguments);
    }

    cipher.doFinal.overload('[B').implementation = function () {
        console.log("Cipher.doFinal('[B') is called!");
        var algorithm = this.getAlgorithm();
        var tag = algorithm + " doFinal data";
        var data = arguments[0];
        toUtf8(tag, data);
        toHex(tag, data);
        toBase64(tag, data);
        var result = this.doFinal.apply(this, arguments);
        var tags = algorithm + " doFinal result";
        toHex(tags, result);
        toBase64(tags, result);
        console.log("=============================");
        return result;
    }
    cipher.doFinal.overload('[B', 'int', 'int').implementation = function () {
        console.log("Cipher.doFinal('[B', 'int', 'int') is called!");
        var algorithm = this.getAlgorithm();
        var tag = algorithm + " doFinal data";
        var data = arguments[0];
        toUtf8(tag, data);
        toHex(tag, data);
        toBase64(tag, data);
        var result = this.doFinal.apply(this, arguments);
        var tags = algorithm + " doFinal result";
        toHex(tags, result);
        toBase64(tags, result);
        console.log("====================", arguments[1], arguments[2]);
        return result;
    }

    var signature = Java.use("java.security.Signature");
    signature.update.overload('byte').implementation = function (data) {
        console.log("Signature.update('byte') is called!");
        return this.update(data);
    }
    signature.update.overload('java.nio.ByteBuffer').implementation = function (data) {
        console.log("Signature.update('java.nio.ByteBuffer') is called!");
        return this.update(data);
    }
    signature.update.overload('[B', 'int', 'int').implementation = function (data, start, length) {
        console.log("Signature.update('[B', 'int', 'int') is called!");
        var algorithm = this.getAlgorithm();
        var tag = algorithm + " update data";
        toUtf8(tag, data);
        toHex(tag, data);
        toBase64(tag, data);
        console.log("=====================================", start, length);
        return this.update(data, start, length);
    }
    signature.sign.overload('[B', 'int', 'int').implementation = function () {
        console.log("Signature.sign('[B', 'int', 'int') is called!");
        return this.sign.apply(this, arguments);
    }
    signature.sign.overload().implementation = function () {
        console.log("Signature.sign() is called!");
        var result = this.sign();
        var algorithm = this.getAlgorithm();
        var tag = algorithm + " sign result";
        toHex(tag, result);
        toBase64(tag, result);
        console.log("=======================================");
        return result;
    }

});
```

### Objection辅助Hook

Objection是一个基于Frida框架的第三方工具包，它实际上做了对Frida框架的进一步封装，通过输入一系列的命令即可完成Hook。输入命令时还可以弹出对应的提示信息，大大降低Frida Hook框架的使用门槛。不过Objection无法对so层代码进行Hook，目前介绍的方法都是对Java层进行Hook。

#### 安装与基本使用

![image-20240307141855761](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081407527.png)

![image-20240307141939293](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081407528.png)

![image-20240307141945174](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081407529.png)

![image-20240307141956156](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081407530.png)