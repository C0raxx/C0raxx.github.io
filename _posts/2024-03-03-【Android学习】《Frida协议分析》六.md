---
layout:     post
title:      【Android学习】《Frida协议分析》六
subtitle:   工具学习
date:       2024-03-03
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【Android应用安全实战：Frida协议分析】
    - 【逆向】
    - 【Android】
---

## 第六章

本章深入JNI函数，具体讲解了如何对系统函数进行Hook及快速定位，通过本章节的学习，读者将会对so层Hook有更加深入的了解。另外，熟练掌握常用工具也是逆向开发者的基本功，除了本章学习的内容之外，希望读者对jnitrace多做了解，多将其应用于实战中。



JNI函数的Hook与快速定位

不管App应用程序自身的so文件如何混淆，系统函数都是不变的。通常可以Hook一系列系统函数来定位关键代码。linker、libc.so、libdl.so、libart.so中都有很多可以被Hook的系统函数，其中libart.so中的JNI函数在so文件开发中很常用。通过Hook JNI函数，可以大体上知晓so函数的代码逻辑，如常用的逆向工具jnitrace就是Hook了大量的JNI函数，并打印参数、返回值及函数栈。本章主要介绍JNI函数的Hook、主动调用及快速定位。

### JNI函数的Hook

本小节将介绍两种获取JNI函数地址并完成Hook的方式。

![image-20240307151853342](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081410479.png)

#### JNIEnv的获取

使用C/C++开发so文件，JNIEnv∗指针变量最终都指向JNINativeInterface结构体。这个结构体中定义了很多函数指针。**而要Hook或者调用这些JNI函数，都需要先获取JNIEnv∗指针变量的内存地址**

![image-20240307151955205](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081410480.png)

使用Java.vm.getEnv()和Java.vm.tryGetEnv()都可以返回Frida包装后的JNIEnv对象。区别是使用getEnv如果没有获取到JNIEnv对象会抛出错误，使用tryGetEnv如果没有获取到JNIEnv对象会返回null。

![image-20240307152010260](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081410481.png)

该对象中的handle属性记录的是原始JNIEnv∗指针变量的内存地址 该变量存放的内容是:f0 de fd a4 6f 00 00 00。字节序转换后的值0x6fa4fddef0就是JNIEnv结构体的地址。

打印出来就是这个样子

![image-20240307152058957](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081410482.png)

JNIEnv结构体中的前4个函数指针是保留的，之后就是一连串的函数指针。env.handle的类型是NativePointer，所以可以使用env.handle.readPointer()来读指针。而env的类型不是NativePointer，不能直接调用readPointer方法，但是可以通过Memory.readPointer(env)来达到相同的效果，也可以通过ptr(env).readPointer()来达到相同的效果。

代码如下

```
console.log(hexdump(Memory.readPointer(env)));
//console.log(hexdump(ptr(env).readPointer()));
/*
6fa4fddef0  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
6fa4fddf00  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
6fa4fddf10  68 32 d6 a4 6f 00 00 00 14 3a d6 a4 6f 00 00 00  h2..o....:..o...
6fa4fddf20  18 42 d6 a4 6f 00 00 00 04 4a d6 a4 6f 00 00 00  .B..o....J..o...
*/
```

要得到原始JNIEnv∗指针变量的内存地址，也可以通过Hook某些函数来做到，如JNI静态注册和动态注册函数的第0个参数就是JNIEnv∗指针变量。

#### 枚举libart符号表来Hook

将libart.so从手机中拉取出来，拖入IDA中反编译。Android 10系统中libart.so所在位置为/system/apex/com.android.runtime.release/lib64/libart.so，

![image-20240307152226567](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081410483.png)

Hook的是不包含CheckJNI的符号

```
function hook_jni() {
    var _symbols = Process.getModuleByName("libart.so").enumerateSymbols();
    var newStringUtf = null;
    for (let i = 0; i < _symbols.length; i++) {
        var _symbol = _symbols[i];
        if(_symbol.name.indexOf("CheckJNI") == -1 && _symbol.name.indexOf("NewStringUTF") != -1){
            newStringUtf = _symbol.address;
        }
    }
    Interceptor.attach(newStringUtf, {
        onEnter: function (args) {
            console.log("newStringUtf  args: ", args[1].readCString());
        }, onLeave: function (retval) {
            console.log("newStringUtf  retval: ", retval);
        }
    });
}
hook_jni();
/*
newStringUtf args:  GB2312
newStringUtf retval:  0x81
newStringUtf args:  41bef1ce7fdc3e42c0e5d940ad74ac00
newStringUtf retval:  0xa9
*/
```

#### 通过计算地址的方式来Hook

![image-20240307152314586](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081410484.png)

Frida框架中可以使用Java.vm.tryGetEnv().handle.readPointer()来得到JNIEnv结构体的地址。NewStringUTF是JNIEnv结构体中的第167个函数指针（从0开始算）。在64位的App应用程序里，一个指针占8个字节，所以从JNIEnv结构体的地址偏移1336个字节（十六进制的0x538）即可得到NewStringUTF函数指针的内存地址。在32位的App应用程序里，一个指针占4个字节，所以从JNIEnv结构体的地址偏移668个字节（十六进制的0x29C）即可得到NewStringUTF函数指针的内存地址。具体代码如下：

```
var envAddr = Java.vm.tryGetEnv().handle.readPointer();
var NewStringUTF = envAddr.add(167 * Process.pointerSize);
var NewStringUTFAddr = envAddr.add(167 * Process.pointerSize).readPointer();
console.log(hexdump(NewStringUTF));
console.log(hexdump(NewStringUTFAddr));
console.log(Instruction.parse(NewStringUTFAddr).toString());
/*
6fa4fde428  ec 30 d7 a4 6f 00 00 00 a0 38 d7 a4 6f 00 00 00  .0..o....8..o...
6fa4fde438  50 40 d7 a4 6f 00 00 00 70 40 d7 a4 6f 00 00 00  P@..o...p@..o...

6fa4d730ec  ff 43 03 d1 fc 6f 07 a9 fa 67 08 a9 f8 5f 09 a9  .C...o...g..._..
6fa4d730fc  f6 57 0a a9 f4 4f 0b a9 fd 7b 0c a9 fd 03 03 91  .W...O...{......

sub sp, sp, #0xd0
*/
```

有了JNI函数的地址，接下来的Hook代码就很容易了。

```
function hook_jni2() {
    var envAddr = Java.vm.tryGetEnv().handle.readPointer();
    var NewStringUTFAddr = envAddr.add(167 * Process.pointerSize).readPointer();
    Interceptor.attach(NewStringUTFAddr, {
        onEnter: function (args) {
            console.log("FindClass args: ", args[1].readCString());
        }, onLeave: function (retval) {
            console.log("FindClass retval: ", retval);
        }
    });
}
hook_jni2();
/*
newStringUtf args:  GB2312
newStringUtf retval:  0x81
newStringUtf args:  41bef1ce7fdc3e42c0e5d940ad74ac00
newStringUtf retval:  0xa9
*/
```

### 主动调用so函数

#### 主动调用so函数

先通过Java.vm.tryGetEnv()获取Frida包装后的JNIEnv对象，接着就可以通过Frida封装的API来调用JNI函数。

```
function hook_jni() {
    var _symbols = Process.getModuleByName("libart.so").enumerateSymbols();
    var newStringUtf = null;
    for (let i = 0; i < _symbols.length; i++) {
        var _symbol = _symbols[i];
        if(_symbol.name.indexOf("CheckJNI") == -1 && _symbol.name.indexOf("NewStringUTF") != -1){
            newStringUtf = _symbol.address;
        }
    }
    Interceptor.attach(newStringUtf, {
        onEnter: function (args) {
            console.log("newStringUtf  args: ", args[1].readCString());
        }, onLeave: function (retval) {
            var cstr = Java.vm.tryGetEnv().getStringUtfChars(retval);
            console.log(hexdump(cstr));
            console.log("newStringUtf  retval: ", cstr.readCString());
        }
    });
}
hook_jni();
/*
newStringUtf  args:  GB2312
6f94653dc0  47 42 32 33 31 32 00 00 00 00 00 00 00 00 00 00  GB2312..........
6f94653dd0  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
newStringUtf  retval:  GB2312

newStringUtf  args:  41bef1ce7fdc3e42c0e5d940ad74ac00
6f96e081d0  34 31 62 65 66 31 63 65 37 66 64 63 33 65 34 32  41bef1ce7fdc3e42
6f96e081e0  63 30 65 35 64 39 34 30 61 64 37 34 61 63 30 30  c0e5d940ad74ac00
newStringUtf  retval:  41bef1ce7fdc3e42c0e5d940ad74ac00
*/
```

主动调用GetStringUTFChars函数，传入的retval参数是jstring类型的，返回的cstr是一个地址。可以通过hexdump函数打印内存，也可以直接调用readCString来显示字符串。最终结果与NewStringUTF传入的实参一致。因此JNI函数GetStringUTFChars和NewStringUTF的作用刚好相反。



#### so层文件打印函数栈

Thread.backtrace来获取函数栈

![image-20240307152700261](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081410485.png)

其中sleep函数比较简单，传入一个参数delay，用于挂起当前线程，在给定的秒数后恢复。backtrace函数用于获取函数栈，传入两个可省略参数，返回NativePointer类型的地址数组。

![image-20240307152733294](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081410486.png)

最好都别省略 可以更加准确一些

想要打印函数栈，只要将Thread.backtrace返回的数组中的地址依次传入DebugSymbol.fromAddress获取相应调试信息后输出即可。

```
function hook_jni() {
    var _symbols = Process.getModuleByName("libart.so").enumerateSymbols();
    var newStringUtf = null;
    for (let i = 0; i < _symbols.length; i++) {
        var _symbol = _symbols[i];
        if(_symbol.name.indexOf("CheckJNI") == -1 && _symbol.name.indexOf("NewStringUTF") != -1){
            newStringUtf = _symbol.address;
        }
    }
    Interceptor.attach(newStringUtf, {
        onEnter: function (args) {
            console.log(Thread.backtrace(this.context, Backtracer.ACCURATE).map(DebugSymbol.fromAddress).join("\n") + "\n");
            console.log("newStringUtf  args: ", args[1].readCString());
        }, onLeave: function (retval) {
        }
    });
}
hook_jni();
/*
0x79ca9793a8 libart.so!_ZN3art12_GLOBAL__N_18CheckJNI12NewStringUTFEP7_JNIEnvPKc+0x2bc
0x79604347f8 libxiaojianbang.so!_ZN7_JNIEnv12NewStringUTFEPKc+0x2c
0x7960434fd0 libxiaojianbang.so!Java_com_xiaojianbang_ndk_NativeHelper_md5+0x194
0x79ca75a354 libart.so!art_quick_generic_jni_trampoline+0x94
0x79ca7515bc libart.so!art_quick_invoke_static_stub+0x23c
......
newStringUtf  args:  41bef1ce7fdc3e42c0e5d940ad74ac00
*/
```

map是JavaScript中数组的方法，用于遍历数组成员

#### DebugSymbol类

![image-20240307152854094](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081410487.png)

简单测试一下这些属性和方法

```
var debsym = DebugSymbol.fromName("strcat");
console.log("address: ", debsym.address);
console.log("name: ", debsym.name);
console.log("moduleName: ", debsym.moduleName);
console.log("fileName: ", debsym.fileName);
console.log("lineNumber: ", debsym.lineNumber);
console.log("toString: ", debsym.toString());

console.log("getFunctionByName: ", DebugSymbol.getFunctionByName("strcat"));
console.log("findFunctionsNamed: ", DebugSymbol.findFunctionsNamed("JNI_OnLoad"));
console.log("findFunctionsMatching: ", DebugSymbol.findFunctionsMatching("JNI_OnLoad"));
/*
address:  0x7a4d20222c
name:  strcat
moduleName:  libc.so
fileName:
lineNumber:  0
toString:  0x7a4d20222c libc.so!strcat

getFunctionByName:  0x7a4d20222c
findFunctionsNamed:  0x79c20cf89c,0x79c206d35c,0x79c08b1898,0x79b6419ab8,0x79b6377014,0x79b62e7070,0x79b27cd1f8,0x796f4eef0c,0x7960434d28
findFunctionsMatching:  0x7960434d28,0x796f4eef0c,0x79b27cd1f8,0x79b62e7070,0x79b6377014,0x79b6419ab8,0x79c08b1898,0x79c206d35c,0x79c20cf89c
*/
```

#### so层主动调用任意函数

Frida提供new NativeFunction的方式来创建函数指针，有了这个函数指针，即可在代码中主动调用so层函数，传入自定义的实参。来看一下NativeFunction的语法：

![image-20240307152933068](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081410488.png)

所以主要就是俩 一个是地址 另外一个是参数

其实传给NativeFunction的返回值和参数类型不用非常精确

实例

![image-20240307153003978](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081410489.png)

```
Java.perform(function () {
    var soAddr = Module.findBaseAddress("libxiaojianbang.so");
    var funAddr = soAddr.add(0x16BC);
    var jstr2cstr = new NativeFunction(funAddr, 'pointer', ['pointer','pointer']);
var env = Java.vm.tryGetEnv();
//主动调用jni函数newStringUtf，将JavaScript的字符串转为Java字符串
    var jstring = env.newStringUtf("xiaojianbang");
    var retval = jstr2cstr(env.handle, jstring);
    //var retval = jstr2cstr(env, jstring);
    console.log(retval.readCString());
});
//xiaojianbang
```

#### 通过NativeFunction主动调用JNI函数

so层主动调用任意函数，既然是任意函数，那自然是包括JNI函数的。本小节就来演示通过NativeFunction声明JNI函数指针调用JNI函数的方法

```
var symbols = Process.getModuleByName("libart.so").enumerateSymbols();
var NewStringUTFAddr = null;
var GetStringUTFCharsAddr = null;
for (var i = 0; i < symbols.length; i++) {
    var symbol = symbols[i];
    if(symbol.name.indexOf("CheckJNI") == -1 && symbol.name.indexOf("NewStringUTF") != -1){
        NewStringUTFAddr = symbol.address;
    }else if (symbol.name.indexOf("CheckJNI") == -1 && symbol.name.indexOf("GetStringUTFChars") != -1){
        GetStringUTFCharsAddr = symbol.address;
    }
}
var NewStringUTF = new NativeFunction(NewStringUTFAddr, 'pointer', ['pointer', 'pointer']);
var GetStringUTFChars = new NativeFunction(GetStringUTFCharsAddr, 'pointer', ['pointer', 'pointer', 'pointer']);

var jstring = NewStringUTF(Java.vm.tryGetEnv().handle, Memory.allocUtf8String("xiaojianbang"));
console.log(jstring);

var cstr = GetStringUTFChars(Java.vm.tryGetEnv(),  jstring,  ptr(0));
console.log(cstr.readCString());
/*
0x1
xiaojianbang
*/


var cstr = GetStringUTFChars(Java.vm.tryGetEnv(),  jstring,  Memory.alloc(1).writeS8(1));
console.log(cstr.readCString());
//xiaojianbang


//......
var GetStringUTFChars = new NativeFunction(GetStringUTFCharsAddr, 'pointer', ['pointer', 'pointer']);
var cstr = GetStringUTFChars(Java.vm.tryGetEnv(),  jstring);
console.log(cstr.readCString());
//xiaojianbang
```

上述代码先获取了NewStringUTF和GetStringUTFChars的函数地址，具体代码已出现过多次，不再赘述。接着使用new NativeFunction声明了两个函数指针，一般参数类型要与jni.h中声明的一致，查看这两个函数在jni.h中的声明：

![image-20240307153106548](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081410490.png)

### JNI函数注册的快速定位

如下一个实例的源码

![image-20240307155943544](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081410491.png)

知道有add encode md5这些native方法 但so加载并非必须在本类当中进行

由此发现要确定native在那个so是一件麻烦耗时间的事情

![image-20240307160029276](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081410492.png)

快速定位的思路也很简单，无非就是Hook系统函数。JNI的函数注册分为静态注册和动态注册，但不管哪一种，都会调用相应的系统函数，找一个合适的函数Hook即可

1. JNI函数静态注册可以Hook dlsym。静态注册的方式在一开始并没有绑定so层的函数。当Java层的native函数首次被调用，系统会按规则构建出对应的函数名，通过dlsym去每一个so文件中寻找符号，找到后进行绑定。
2. JNI函数动态注册可以Hook RegisterNatives。这个很容易理解，因为就是使用这个函数来动态注册的。当然也可以对系统源码进行分析，找一个更底层的函数来Hook。
3. Hook其他JNI函数，如NewStringUTF函数，然后打印函数栈即可。也可以直接使用jnitrace

#### Hook dlsym获取函数地址

![image-20240307160216563](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081410493.png)

handle是使用dlopen函数之后返回的句柄，symbol是要求获取的函数的名称，返回值是void∗，指向函数的地址。

hook

```
var dlsymAddr = Module.findExportByName("libdl.so", "dlsym");
Interceptor.attach(dlsymAddr, {
    onEnter: function (args) {
        this.args1 = args[1];
    }, onLeave: function (retval) {
        console.log(this.args1.readCString(),  retval);
    }
});
/*
//app以spawn的方式启动，得到如下输出
oatdata 0x7d360c3000
......
oatdatabimgrelro 0x0
HMI 0x7d33c94018
//点击CADD按钮后输出，之后不再触发
JNI_OnLoad 0x7d337abe10
Java_com_xiaojianbang_ndk_NativeHelper_add 0x7d337abacc
//点击CMD5按钮后输出，之后不再触发
Java_com_xiaojianbang_ndk_NativeHelper_md5 0x7d337abf2c
*/
```

dlsym函数返回的地址有可能为0。静态注册的native函数首次被调用才会经过dlsym函数，之后不再触发。JNI_OnLoad不属于静态注册函数，只是系统在so文件加载完毕后，去获取JNI_OnLoad函数地址，使用的也是dlsym。

#### Hook RegisterNatives获取函数地址

要Hook一个函数，必然要先得到函数地址，RegisterNatives定义在libart.so中

```
var RegisterNativesAddr = null;
var _symbols = Process.findModuleByName("libart.so").enumerateSymbols();
for (var i = 0; i < _symbols.length; i++) {
    var _symbol = _symbols[i];
    if (_symbol.name.indexOf("CheckJNI") == -1 && _symbol.name.indexOf("RegisterNatives") != -1) {
        RegisterNativesAddr = _symbols[i].address;
    }
}
console.log(RegisterNativesAddr);
// 0x7da0a0a158
```

查看RegisterNatives和JNINativeMethod在jni.h中的声明

![image-20240307160604731](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081410494.png)

RegisterNatives有4个参数，第0个参数是JNIEnv∗，第1个参数表示注册的函数是哪个类的，**第2个参数是JNINativeMethod结构体地址**，第3个参数是动态注册的函数个数。

![image-20240307160645022](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081410495.png)

这是注册函数为一个的时候，实际上注册多个函数的时候JNINativeMethod是一个结构体数组。数组成员的个数就是注册函数的个数。每个成员占3个指针长度。因此，上述代码还需要加一个循环，每循环一次，再偏移3个指针长度。最终代码如下

```
var RegisterNativesAddr = null;
var _symbols = Process.findModuleByName("libart.so").enumerateSymbols();
for (var i = 0; i < _symbols.length; i++) {
    var _symbol = _symbols[i];
    if (_symbol.name.indexOf("CheckJNI") == -1 && _symbol.name.indexOf("RegisterNatives") != -1) {
        RegisterNativesAddr = _symbols[i].address;
    }
}

Interceptor.attach(RegisterNativesAddr, {
    onEnter: function (args) {
        var env = Java.vm.tryGetEnv();
        var className = env.getClassName(args[1]);
        var methodCount = args[3].toInt32();

        for (let i = 0; i < methodCount; i++) {
            var methodName = args[2].add(Process.pointerSize * 3 * i)
.readPointer().readCString();
            var signature = args[2].add(Process.pointerSize * 3 * i)
.add(Process.pointerSize).readPointer().readCString();
            var fnPtr = args[2].add(Process.pointerSize * 3 * i)
.add(Process.pointerSize * 2).readPointer();
            var module = Process.findModuleByAddress(fnPtr);
            console.log(className, methodName, signature, fnPtr, module.name, fnPtr.sub(module.base));
        }
    }, onLeave: function (retval) {
    }
});
```

### ollvm混淆应用协议分析实战

主要是应用到一个工具 jnitrace

#### jnitrace工具的使用

而jnitrace就是Hook一系列**JNI函数**的工具。

#### 实战：某App应用程序协议分析

`jnitrace-m attach-l∗com.xxxx.android`

![image-20240307161130121](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081410496.png)

上述输出内容中，显示了被Hook的函数名、参数和返回值，C语言的数据类型会显示对应内存中的数据，Java的数据类型不作显示。还打印了函数栈信息，Backtrace中显示了上层函数地址、上层函数名、当前函数相对上层函数的偏移量、模块名、模块基址。而且从第1个参数输出的内容中，可以看出这是RSA算法的公钥。

![image-20240307161149251](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081410497.png)

从上述输出中可以看出，调用了Java的Cipher类的doFinal方法。