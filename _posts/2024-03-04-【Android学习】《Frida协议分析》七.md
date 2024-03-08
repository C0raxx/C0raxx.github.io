---
layout:     post
title:      【Android学习】《Frida协议分析》七
subtitle:   工具学习
date:       2024-03-04
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【Android应用安全实战：Frida协议分析】
    - 【逆向】
    - 【Android】
---

## 第七章

本章讲解了Frida操作内存数据、常用的NativePointer类、Memory的常用方法，以及替换函数，最后讲解了更重要的一些系统函数的Hook，也实战了一个小案例。本章虽然带领读者学习了Frida框架的一些进阶内容，但这是远远不够的，任何工具的学习，从基础到进阶都是较为困难的，真正的进阶知识需要在实践中领悟。



**Frida框架so层进阶应用**

包括Frida操作内存数据、Frida常用API和Frida进阶Hook等。

### Frida操作内存数据

Frida框架如何操作内存数据，包括内存读写、修改so函数代码、从内容中导出so函数、ollvm字符串解密、构造二级指针和读写文件。

#### 内存读写

##### 修改内存权限

先使用Memory的protect方法来修改对应内存区域的权限。

源码

![image-20240307161506292](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081411043.png)

protect方法需要传入3个参数，第0个参数是指定内存地址，第1个参数是需要修改的内存大小，第2个参数是string类型的所需内存权限。

##### 读取指定地址的字符串

计算出指定字符串地址后，可以使用NativePointer的readCString方法来读取字符串 乘上

```
var soAddr = Module.findBaseAddress("libxiaojianbang.so");
console.log(soAddr.add(0x3DED).readCString());
// com/xiaojianbang/ndk/NativeHelper
```

##### 导出指定地址的内存数据

用hexdump

```
var soAddr = Module.findBaseAddress("libxiaojianbang.so");
console.log(hexdump(soAddr.add(0x3DED)));
/*
795b345ded  63 6f 6d 2f 78 69 61 6f 6a 69 61 6e 62 61 6e 67  com/xiaojianbang
795b345dfd  2f 6e 64 6b 2f 4e 61 74 69 76 65 48 65 6c 70 65  /ndk/NativeHelpe
795b345e0d  72 00 65 6e 63 6f 64 65 00 28 29 4c 6a 61 76 61  r.encode.()Ljava
*/
```

##### 读取指定地址的内存数据

计算出指定内存地址后，可以使用NativePointer的readByteArray方法读取一段内存数据，该方法很常用，如Frida导出so文件的基本原理就是从so文件基址开始，读取整个so文件大小的数据，然后写文件保存。如果要从指定地址开始，读取16字节的内存数据，代码如下：

```
var soAddr = Module.findBaseAddress("libxiaojianbang.so");
console.log(soAddr.add(0x3DED).readByteArray(16));
/*
00000000  63 6f 6d 2f 78 69 61 6f 6a 69 61 6e 62 61 6e 67  com/xiaojianbang
*/
```

##### 写入数据到指定内存地址

计算出指定内存地址后，可以使用NativePointer的writeByteArray方法，从该地址开始往后写入数据

![image-20240307161727068](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081411044.png)

```
// 将字符串转为字节数组
function stringToBytes(str){
    return hexToBytes(stringToHex(str));
}
// 将字符串进行hex编码
function stringToHex(str) {
    return str.split("").map(function(c) {
        return ("0" + c.charCodeAt(0).toString(16)).slice(-2);
    }).join("");
}
// 将hex编码的数据转为字节数组
function hexToBytes(hex) {
    for (var bytes = [], c = 0; c < hex.length; c += 2)
        bytes.push(parseInt(hex.substr(c, 2), 16));
    return bytes;
}
// 将hex编码的数据转为字符串
function hexToString(hexStr) {
    var hex = hexStr.toString();
    var str = '';
    for (var i = 0; i < hex.length; i += 2)
        str += String.fromCharCode(parseInt(hex.substr(i, 2), 16));
    return str;
}
```

向该地址写入字符串xiaojianbang

```
var soAddr = Module.findBaseAddress("libxiaojianbang.so");
var tmpAddr = soAddr.add(0x3DED);
Memory.protect(tmpAddr, 16, 'rwx');
console.log(hexdump( tmpAddr.writeByteArray(stringToBytes("xiaojianbang\0")) ));
/*
795b345ded  78 69 61 6f 6a 69 61 6e 62 61 6e 67 00 64 65 66  xiaojianbang.def
*/
```

如果要求将这段数据hex编码 则

![image-20240307161850071](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081411045.png)

##### 分配内存

有些时候主动调用一些so层函数的时候需要自己构建新数据传参 这个时候需要分配内存 调用memory的alloc分配

![image-20240307161942255](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081411046.png)

Memory中的alloc函数会在Frida私有堆上分配指定大小的内存，返回NativePointer类型的首地址。接着用NativePointer类的writeByteArray方法写入内存即可。

```
var addr = Memory.alloc(8);
addr.writeByteArray(hexToBytes("eeeeeeeeeeeeeeee"));
console.log(addr.readByteArray(8));
/*
00000000  ee ee ee ee ee ee ee ee                          ........
*/
```

##### Frida修改so函数代码

某函数在so文件中的偏移地址为0x1ACC，传入的5个实参分别为JNIEnv∗、jclass、5、6、7，返回结果为18

我们读取一下这段so代码

```
var funcAddr = Module.findBaseAddress("libxiaojianbang.so").add(0x1ACC);
var opcodes = funcAddr.readByteArray(16);
console.log(opcodes);
/*
00000000  ff 83 00 d1 e0 0f 00 f9 e1 0b 00 f9 e2 0f 00 b9  ................
*/
```

可以看到和汇编一致

![image-20240307162154101](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081411047.png)

frida 当中也内置函数把他转成汇编

![image-20240307162222329](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081411048.png)

##### 通过修改内存中的opcode来修改函数代码

直接修改opcode

需要arm和opcode转换

```
var soAddr = Module.findBaseAddress("libxiaojianbang.so");
soAddr.add(0x1AF4).writeByteArray(hexToBytes("0001094B"));
console.log(Instruction.parse(soAddr.add(0x1AF4)));
// sub w0, w8, w9
```

```
// 通常是进入函数的第一步，提升函数栈空间，需要16字节对齐
SUB  SP, SP, #0x20
// 也是arm汇编进入函数的基本操作，保存寄存器中的参数到栈中
// 内存中的数据CPU无法直接运算，会使用str/ldr来操作数据
// str用于将寄存器中的值保存到内存中，ldr用于将内存中的值加载到寄存器中
// x0、x1对应实参JNIEnv*、jclass，因为是指针，所以用64位寄存器
// w2、w3、w4分别对应实战5、6、7，源码中定义的类型是int，所以用32位寄存器
STR  X0, [SP,#0x20+var_8]
STR  X1, [SP,#0x20+var_10]
STR  W2, [SP,#0x20+var_14]
STR  W3, [SP,#0x20+var_18]
STR  W4, [SP,#0x20+var_1C]
// var_14中的值加载到w8中，var_14之前保存了w2的值5，这一句相当于w8 = 5
LDR  W8, [SP,#0x20+var_14]
// var_18中的值加载到w9中，var_18之前保存了w3的值6，这一句相当于w9 = 6
LDR  W9, [SP,#0x20+var_18]
// w8 = w8 + w9  =>  w8 = 11
ADD  W8, W8, W9
// var_1C中的值加载到w9中，var_1C之前保存了w4的值7，这一句相当于w9 = 7
LDR  W9, [SP,#0x20+var_1C]
// w0 = w8 + w9  =>  w0 = 11 + 7  =>  w0 = 18
// arm64中，函数返回值存放于w0/x0
ADD  W0, W8, W9
// 释放栈空间
ADD  SP, SP, #0x20
// 相当于bl  lr，返回
RET
```

现在相当于变成了减法

##### 使用Frida提供的API来写汇编代码

将偏移地址0x1AEC处的指令改为nop

```
var soAddr = Module.findBaseAddress("libxiaojianbang.so");
new Arm64Writer(soAddr.add(0x1AEC)).putNop();
console.log(Instruction.parse(soAddr.add(0x1AEC)).toString());
// nop
```

还可以用Memory的patchCode方法

第0个参数是修改的起始地址，第1个参数是修改的字节数，第2个参数是回调函数，当执行到起始地址时，会调用回调函数。如果要将偏移地址0x1AF4处的ADD w0, w8, w9修改为SUB w0,w8, w9，实现修改的代码如下：

```
var codeAddr = Module.findBaseAddress("libxiaojianbang.so").add(0x1AF4);
Memory.patchCode(codeAddr, 4, function (code) {
    var writer = new Arm64Writer(code, { pc: codeAddr });
    writer.putBytes(hexToBytes("0001094B"));   //sub w0, w8, w9
    writer.flush();
});
```

Frida在进行so层Hook时，需要修改函数的前16个字节，这也是函数被Hook的检测点之一。可以通过打印Hook前后的函数首地址上的内存数据来验证 –防检测

```
function hook_func() {
    var soAddr = Module.findBaseAddress("libxiaojianbang.so");
    var MD5Final = soAddr.add(0x3A78);
    console.log(hexdump(MD5Final.readByteArray(20)));
    Interceptor.attach(MD5Final, {
        onEnter: function (args) {
            console.log(hexdump(MD5Final.readByteArray(20)));
        }, onLeave: function (retval) {
        }
    });
}
hook_func();
/*
00000000  ff 43 01 d1 fd 7b 04 a9 fd 03 01 91 48 d0 3b d5  .C...{......H.;.
00000010  08 15 40 f9                                      ..@.

00000000  50 00 00 58 00 02 1f d6 00 96 33 3e 74 00 00 00  P..X......3>t...
00000010  08 15 40 f9                                      ..@.
*/
```

#### Frida从内存中导出so函数

某些App应用程序会对so文件进行加固，或者对so函数中的字符串进行加密。而加载到内存中的so函数，一般是解密状态，这时就可以使用之前小节中介绍的内存读写方法，将so函数从内存中保存下来

```
function dump_so(so_name) {
    Java.perform(function () {
        var module = Process.getModuleByName(so_name);
        console.log("[name]:", module.name);
        console.log("[base]:", module.base);
        console.log("[size]:", module.size);
        console.log("[path]:", module.path);
        var currentApplication = Java.use("android.app.ActivityThread").currentApplication();
        var dir = currentApplication.getApplicationContext().getFilesDir().getPath();
        var path = dir + "/" + module.name + "_" + module.base + "_" + module.size + ".so";
        var file = new File(path, "wb");
        if (file) {
            Memory.protect(module.base, module.size, 'rwx');
            var buffer = module.base.readByteArray(module.size);
            file.write(buffer);
            file.flush();
            file.close();
            console.log("[dump]:", path);
        }
    });
}
dump_so("libxiaojianbang.so");
/*
[name]: libxiaojianbang.so
[base]: 0x74c6c39000
[size]: 28672
[path]: /data/app/com.xiaojianbang.app-Kykbukopl-edrrBKaPhfyg==/lib/arm64/libxiaojianbang.so
[dump]: /data/user/0/com.xiaojianbang.app/files/libxiaojianbang.so_0x74c6c39000_28672.so
*/
```

通过传入的模块名，找到对应的Module，得到模块基址、模块大小等必要信息。然后生成保存路径，该路径最好是App应用程序本身的私有目录，防止因为权限问题而保存失败。接着使用Memory.protect修改内存权限，再使用NativePointer的readByteArray方法读取整个so文件对应的内存数据。最后使用frida写文件的API将文件写出。

#### ollvm字符串解密

被加密的字符串

![image-20240307163548049](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081411049.png)

运行起来之后都会被解密

##### 直接打印内存中的字符串

```
var soAddr = Module.findBaseAddress("liblogin_encrypt.so");
console.log(soAddr.add(0xD060).readCString());
// java/security/KeyFactory

```

可以看到内存中的字符串是解密状态，这种方式比较烦琐。

##### 使用jnitrace

只能查看JNI相关函数

##### 从内存中导出整个so文件

直接从内存中保存下来的so文件放到IDA中逆向分析是会报错的，需要修复。

修复后面再学 这本书里面没讲

![image-20240307163807526](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081411050.png)

#### 构造二级指针

二级指针

![image-20240307163930568](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081411051.png)

将strAddr存入到指针变量中，就构建出二级指针了。代码如下

![image-20240307164016240](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081411052.png)

#### 读写文件

写出文件

使用fopen打开文件，fputs写入数据，fclose关闭文件。

```
var fopenAddr = Module.findExportByName("libc.so", "fopen");
var fputsAddr = Module.findExportByName("libc.so", "fputs");
var fcloseAddr = Module.findExportByName("libc.so", "fclose");
```

使用new NativeFunction来声明函数指针

```
// FILE *fopen(const char *filename, const char *mode)
// int fputs(const char *str, FILE *stream)
// int fclose(FILE *stream)
var fopen = new NativeFunction(fopenAddr, "pointer", ["pointer", "pointer"]);
var fputs = new NativeFunction(fputsAddr, "int", ["pointer", "pointer"]);
var fclose = new NativeFunction(fcloseAddr, "int", ["pointer"]);

```

写出到App应用程序的私有目录不需要申请权限，其他目录需要App应用程序具有相应权限才能成功。具体实现代码如下：

```
var fileName = Memory.allocUtf8String("/data/data/com.xiaojianbang.app/xiaojianbang.txt");
var openMode = Memory.allocUtf8String("w");
var buffer = Memory.allocUtf8String("QQ24358757");

var file = fopen(filename, open_mode);
fputs(buffer, file);
fclose(file);
```

fopen和fputs接收的都是pointer参数，不能将JS的字符串直接传递过去，需要使用Memory的allocUtf8String在内存中构建相应字符串后，再传递地址。

### Frida其他常用API介绍

包括NativePointer类的常用方法、Memory的常用方法和替换函数。

#### NativePointer类的常用方法

```
declare class NativePointer {
    // NativePointer构造函数，通过new NativePointer(...) 或者ptr(...) 来使用
constructor(v: string | number | UInt64 | Int64 | NativePointerValue);
// 判断是否空指针
isNull(): boolean;
// 创建新指针，值等于this + v
add(v: NativePointerValue | UInt64 | Int64 | number | string): NativePointer;
// 创建新指针，值等于this - v
    sub(v: NativePointerValue | UInt64 | Int64 | number | string): NativePointer;
// 更多用于指针计算的方法，请自行查阅相关文档和源码
......	
// 比较两者是否相等
equals(v: NativePointerValue | UInt64 | Int64 | number | string): boolean;
// 比较两者大小，返回1、-1、0
compare(v: NativePointerValue | UInt64 | Int64 | number | string): number;
// 指针转32位有符号数
toInt32(): number;
// 指针转32位无符号数
toUInt32(): number;
// NativePointer类型转string类型，参数可以指定进制，默认16进制
toString(radix?: number): string;
    toJSON(): string;

// 读取4/8字节数据，转指针
readPointer(): NativePointer;
// 读8bit数据，也就是1字节数据，转有符号数
readS8(): number;
// 读8bit数据，也就是1字节数据，转无符号数
readU8(): number;
// 更多读取数据转数值的方法依此类推
......	
// 读指定字节数内存数据，返回ArrayBuffer 
readByteArray(length: number): ArrayBuffer | null;
// 读指定长度的C语言char*字符串，或者读取到遇字节0为止
readCString(size?: number): string | null;
// 读指定长度的Utf8字符串（可以是中文），或者读取到遇字节0为止
readUtf8String(size?: number): string | null;
readUtf16String(length?: number): string | null;
// 仅限Windows平台使用
    readAnsiString(size?: number): string | null;

// 将4/8字节指针写入内存，后续基本都是与读取类似的操作，不再赘述
writePointer(value: NativePointerValue): NativePointer;
    writeS8(value: number | Int64): NativePointer;
    writeU8(value: number | UInt64): NativePointer;
......
    writeByteArray(value: ArrayBuffer | number[]): NativePointer;
    writeUtf8String(value: string): NativePointer;
writeUtf16String(value: string): NativePointer;
    writeAnsiString(value: string): NativePointer;
}
```

#### Memory的常用方法

```
declare namespace Memory {
// 异步，在内存中搜索指定数据
// 可以搜索指定指令，然后patch。可以搜索指定文件头，然后dump等等
    function scan(address: NativePointerValue, size: number | UInt64, pattern: string, callbacks: MemoryScanCallbacks): void;
// scan的同步版本
    function scanSync(address: NativePointerValue, size: number | UInt64, pattern: string): MemoryScanMatch[];
// 在frida私有堆上分配指定大小的内存，返回首地址
    function alloc(size: number | UInt64, options?: MemoryAllocOptions): NativePointer;
// 在frida私有堆上，将str作为UTF-8字符串进行分配、编码和写出
//（可以是中文）
    function allocUtf8String(str: string): NativePointer;
    function allocUtf16String(str: string): NativePointer;
// 仅限Windows平台使用
    function allocAnsiString(str: string): NativePointer;
// 内存拷贝，参数为目标地址、源地址、要复制的字节数
    function copy(dst: NativePointerValue, src: NativePointerValue, n: number | UInt64): void;
// 先分配内存，然后进行拷贝
    function dup(address: NativePointerValue, size: number | UInt64): NativePointer;
// 修改内存页权限，之前小节中有介绍，这里不再赘述
    function protect(address: NativePointerValue, size: number | UInt64, protection: PageProtection): boolean;
// 可以用来patch指令，本书后续内容中单独介绍
    function patchCode(address: NativePointerValue, size: number | UInt64, apply: MemoryPatchApplyCallback): void;
}
```

#### 替换函数

Interceptor.replace

```
declare namespace Interceptor {
    function replace(target: NativePointerValue, replacement: NativePointerValue,
        data?: NativePointerValue): void;
}
```

replace函数的第0个参数需要传入NativePointer类型的目标地址，第1个参数需要传入用于替换的函数，但也得是NativePointer类型的。在实际应用中，第1个参数通常由new NativeCallback来创建。

```
declare class NativeCallback extends NativePointer {
constructor(func: NativeCallbackImplementation, retType: NativeType, argTypes:
 NativeType[], abi?: NativeABI);
}
```

NativeCallback在实例化时，需要传入函数的实现、返回值类型、参数类型数组。

实例

```
function hook_func() {
    var md5Func = Module.findBaseAddress("libxiaojianbang.so").add(0x1F2C);
    Interceptor.replace(md5Func, new NativeCallback(function () {
        
    }, "void", [ ]));
}
hook_func();
//logcat中的输出为
//CMD5 md5Result: null
```

如果新函数影响了原函数的代码逻辑 应用程序可能崩溃 修改

```
function hook_func() {
    var md5Func = Module.findBaseAddress("libxiaojianbang.so").add(0x1F2C);
    Interceptor.replace(md5Func, new NativeCallback(function () {
        return 100;
    }, "int", []));
}
hook_func();
//该函数触发后，app崩溃
```

### Frida进阶Hook

#### Hook系统函数dlopen

Hook dlopen的必要性：

例如一个so里面的init，函数在JNIOnloa中被调用 而JNI_OnLoad函数比较特殊，so文件被加载以后，系统会自动调用JNI_OnLoad函数。因此，myInit函数在libxiaojianbang.so加载后，就会被调用。

![image-20240307170056132](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081411053.png)

如果spawn注入js so还没加载 会找不到so基址而报错。

如果attach注入js so已经加载晚了 想要hook的函数已经执行完毕

所以这不是js注入时机的问题

要成功Hook这个函数，需要一个良好的Hook时机。

1. 需要在so文件加载之后Hook，因为so文件加载以后才能获取到so文件基址。·需要在myInit函数调用之前，或者JNI_OnLoad调用之前，因为该函数有可能会放在JNI_OnLoad函数中的第一行。
2. 需要在so文件加载之后Hook，因为so文件加载以后才能获取到so文件基址。·需要在myInit函数调用之前，或者JNI_OnLoad调用之前，因为该函数有可能会放在JNI_OnLoad函数中的第一行。

dlopen这个系统函数刚好满足需求，系统加载的so文件和自己加载的so文件，一般都要通过这个函数。

dlopen的第0个参数是被加载的so文件所在全路径，第1个参数用于指定加载模式。

```
function hook_func() {
    var myInit = Module.findBaseAddress("libxiaojianbang.so").add(0x1DE8);
    Interceptor.replace(myInit,  new NativeCallback(function () {
        console.log("replace myInit success");
    }, "void",  [ ]));
}
function hook_dlopen() {
    var android_dlopen_ext = Module.findExportByName("libdl.so", "android_dlopen_ext");
    Interceptor.attach(android_dlopen_ext, {
        onEnter: function (args) {
            var soPath = args[0].readCString();
            if(soPath.indexOf("libxiaojianbang.so") != -1) this.hook = true;
        }, onLeave: function (retval) {
            if(this.hook)  hook_func();
        }
    });
}
hook_dlopen();
// replace myInit success
```

`frida-U-f com.xiaojianbang.app-no-pause-l sohook.js`

在onEnter处判断正在加载的so文件全路径包含libxiaojianbang.so字符串，则在onLeave处执行hook代码。android_dlopen_ext是高版本Android系统中用于加载so文件的函数。该函数执行完毕，so文件就已经加载到内存中，而JNI_OnLoad尚未调用。此时可以获取到so文件基址，完成相应函数的Hook。

低版本用于加载so文件的函数是dlopen 优化

```
function hook_func() {
    var myInit = Module.findBaseAddress("libxiaojianbang.so").add(0x1DE8);
    Interceptor.replace(myInit,  new NativeCallback(function () {
        console.log("replace myInit success");
    }, "void",  [ ]));
}
function hook_dlopen(addr, soName, callback) {
    Interceptor.attach(addr, {
        onEnter: function(args){
            var name = args[0].readCString();
            if(name.indexOf(soName) != -1) this.hook = true;
        }, onLeave: function(retval){
            if(this.hook) callback();
        }
    });
}
var dlopen = Module.findExportByName("libdl.so", "dlopen");
var android_dlopen_ext = Module.findExportByName("libdl.so", "android_dlopen_ext");
hook_dlopen(dlopen, "libxiaojianbang.so", hook_func);
hook_dlopen(android_dlopen_ext, "libxiaojianbang.so", hook_func);
// replace myInit success
```

#### Hook系统函数JNI_Onload

JNI_OnLoad虽然比较特殊，在so文件加载后，由系统自动调用。但是JNI_OnLoad也是在dlopen之后调用的，所以Hook方法与普通函数一致，Hook dlopen即可。

```
function hook_JNIOnload() {
    var JNI_OnLoad = Module.findExportByName("libxiaojianbang.so", "JNI_OnLoad");
    Interceptor.attach(JNI_OnLoad, {
        onEnter: function(args){
            console.log(args[0]);
        }, onLeave: function(retval){
            console.log(retval);
        }
    });
}
function hook_dlopen(addr, soName, callback) {
    Interceptor.attach(addr, {
        onEnter: function(args){
            var name = args[0].readCString();
            if(name.indexOf(soName) != -1) this.hook = true;
        }, onLeave: function(retval){
            if(this.hook) callback();
        }
    });
}
var dlopen = Module.findExportByName("libdl.so", "dlopen");
var android_dlopen_ext = Module.findExportByName("libdl.so", "android_dlopen_ext");
hook_dlopen(dlopen, "libxiaojianbang.so", hook_JNIOnload);
hook_dlopen(android_dlopen_ext, "libxiaojianbang.so", hook_JNIOnload);
/*
0x7a4faaf1c0
0x10006		// JNIOnload函数必须返回一个合理的jni版本号，这里返回的是1.6
*/ 
```

#### Hook系统函数initarray

在so文件加载过程中，系统会自动调用so文件中定义的init、init_array和JNI_OnLoad函数。其中init和init_array是在dlopen函数执行过程中调用的，JNI_OnLoad是在dlopen函数执行之后调用的。

要Hook init和init_array，只Hook dlopen是做不到的。在dlopen的onEnter函数中，so文件还没有加载，无法Hook so文件中的init和init_array。

就需要在dlopen函数内部再找一个Hook点。而这个函数必须满足以下需求，该函数执行之前so文件已经加载完毕，且so文件中的init和init_array尚未被调用。linker中的call_constructors函数满足这些需求

```
function hook_dlopen(addr, soName, callback) {
    Interceptor.attach(addr, {
        onEnter: function (args) {
            var soPath = args[0].readCString();
            if(soPath.indexOf(soName) != -1) callback();
        }, onLeave: function (retval) {
        }
    });
}
var dlopen = Module.findExportByName("libdl.so", "dlopen");
var android_dlopen_ext = Module.findExportByName("libdl.so", "android_dlopen_ext");
hook_dlopen(dlopen, "libxiaojianbang.so", hook_call_constructors);
hook_dlopen(android_dlopen_ext, "libxiaojianbang.so", hook_call_constructors);

```

在linker64中枚举符号，代码如下

```
function hook_call_constructors() {
    var _symbols = Process.getModuleByName("linker64").enumerateSymbols();
    var call_constructors_addr = null;
    for (let i = 0; i < _symbols.length; i++) {
        var _symbol = _symbols[i];
        if(_symbol.name.indexOf("call_constructors") != -1){
            call_constructors_addr = _symbol.address;
        }
    }
    Interceptor.attach(call_constructors_addr, {
        onEnter: function (args) {
        	hook_initarray();
        }, onLeave: function (retval) {
        }
    });
}
```

，hook_initarray函数才是具体的业务逻辑代码，其他代码都是为了hook_initarray能够成功执行而做的一系列铺垫。

```
function hook_initarray(){
    var xiaojianbangAddr = Module.findBaseAddress("libxiaojianbang.so");
    var func1_addr = xiaojianbangAddr.add(0x1D14);
    var func2_addr = xiaojianbangAddr.add(0x1D3C);
    var func3_addr = xiaojianbangAddr.add(0x1CEC);
    Interceptor.replace(func1_addr, new NativeCallback(function () {
        console.log("func1 is replaced!!!");
    }, 'void', []));
    Interceptor.replace(func2_addr, new NativeCallback(function () {
        console.log("func2 is replaced!!!");
    }, 'void', []));
    Interceptor.replace(func3_addr, new NativeCallback(function () {
        console.log("func3 is replaced!!!");
    }, 'void', []));
    Interceptor.detachAll();
}
```

#### Hook系统函数pthread_create

这也是比较常规的逆向思路

一些用于检测的函数通常需要实时运行，那么就有可能用pthread_create开启一个子线程。可以通过Hook这个函数来查看App应用程序为哪些函数开启了线程，就可以有针对性地去分析这些函数的代码逻辑是否与检测相关。

第0个参数为指向线程标识符的指针，第1个参数用来设置线程属性，第2个参数是线程运行函数的起始地址，最后1个参数是传递给线程运行函数的参数。

![image-20240307171014915](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081411054.png)

```
var pthread_create_addr = Module.findExportByName("libc.so", "pthread_create");
Interceptor.attach(pthread_create_addr,{
    onEnter:function(args){
        console.log(args[0], args[1], args[2], args[3]);
        var Module = Process.findModuleByAddress(args[2]);
        if(Module != null) console.log(Module.name, args[2].sub(Module.base));
    },onLeave:function(retval){
    }
});
/*
0x7fc2d0bc18 0x7fc2d0bc20 0x7e21962f90 0x7e259783c0
libutils.so 0x12f90
......
0x7da0bfe9a0 0x0 0x7da097b8dc 0x7e259cbe00
libart.so 0x34b8dc
......
0x7fc2d09378 0x0 0x7d91152d8c 0x0
libxiaojianbang.so 0x1d8c
*/
```

上述代码输出了pthread_create的4个参数，并根据第2个参数的地址寻找对应模块，取得模块名和函数对应so文件中的偏移地址。对于pthread_create的Hook其实不需要做太多操作，只是用来提示App应用程序为这些函数开启了线程，然后有针对性的替换函数即可。

#### 监控内存读写

用来查看哪个函数首次访问了这块内存。

使用Process.setExceptionHandler(callback)可以设置异常处理回调函数。当异常触发时，会调用该回调函数。在该回调函数中，可以通过修改寄存器和内存来让程序从异常中恢复。如果处理了异常，函数最后需要返回true，Frida会立即恢复线程。

思路很简单，先通过Frida API设置异常处理回调，再修改某一块内存区域的访问权限。

```
function hook_dlopen(addr, soName, callback) {
    Interceptor.attach(addr, {
        onEnter: function (args) {
            var soPath = args[0].readCString();
            if(soPath.indexOf(soName) != -1) this.hook = true;
        }, onLeave: function (retval) {
            if (this.hook) {callback()}
        }
    });
}
var dlopen = Module.findExportByName("libdl.so", "dlopen");
var android_dlopen_ext = Module.findExportByName("libdl.so", "android_dlopen_ext");
hook_dlopen(dlopen, "libxiaojianbang.so", set_read_write_break);
hook_dlopen(android_dlopen_ext, "libxiaojianbang.so", set_read_write_break);

function set_read_write_break(){
    Process.setExceptionHandler(function(details) {
console.log(JSON.stringify(details, null, 2));
        Memory.protect(details.memory.address, Process.pointerSize, 'rwx');
        return true;
    });
    var addr = Module.findBaseAddress("libxiaojianbang.so").add(0x3DED);
    Memory.protect(addr, 8, '---');
}
/*
{
  "message": "access violation accessing 0x75ecd31e6b",
  "type": "access-violation",
  "address": "0x767e9e75c4",
  "memory": {
    "operation": "read",
    "address": "0x75ecd31e6b"
  },
  "context": {
    "pc": "0x767e9e75c4",
    "sp": "0x7fc2481e50",
    "x0": "0x7681a52188",
    "x1": "0x75ecd31e6b",
......
    "lr": "0x767e9e73f8"
  },
  "nativeContext": "0x7fc2480c70"
}
*/
```

由于Memory.protect修改的是内存页的权限，并不是只修改给定字节数的权限，所以会存在误差，导致监控内存读写的位置不是很精确。监控内存读写最推荐的是使用unidbg，这是一个基于unicorn开发的，能够在PC端模拟执行so文件的框架。

#### 函数追踪工具frida-trace

先来介绍一个IDA插件trace_natives。使用该插件获取so文件代码段中所有函数的偏移地址后，再配合frida-trace就可以打印函数内部调用流程。

会生成一个txt文件，里面记录了so文件代码段中汇编指令大于10行的函数偏移地址 也可以在py里修改这个10

![image-20240307171312465](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081411055.png)

**trace_natives用来生成frida-trace运行所需参数，而真正的Hook由frida-trace来完成。**

```
frida-trace -UF -O C:\Users\Administrator\Desktop\libxiaojianbang_1634124936.txt

Instrumenting...
sub_16bc: Auto-generated handler at "D:\\Project\\JSProject\\HookProject\\src\\__handlers__\\libxiaojianbang.so\\sub_16bc.js"
......
Started tracing 25 functions. Press Ctrl+C to stop.
           /* TID 0x27ac */
 59084 ms  sub_1f2c()
 59084 ms     | sub_16bc()
 59084 ms     |    | sub_1854()
 ......
 59085 ms     | sub_2230()
 59085 ms     | sub_22a0()
 59085 ms     | sub_3a78()
 59085 ms     |    | sub_3b74()
 59085 ms     |    | sub_22a0()
 59085 ms     |    | sub_22a0()
 59086 ms     |    |    | sub_2518()
 59086 ms     |    |    |    | sub_3cb0()
 59086 ms     |    | sub_3b74()
 59086 ms     | sub_20f4()
 ......
 59086 ms     | sub_21f0()
 59087 ms     | sub_188c()
```

frida-trace会为每个被Hook的函数生成对应的JavaScript文件，并保存在当前目录下的__handlers__/libxiaojianbang.so/目录下，然后显示开始trace函数的数量。当函数被触发后，会按上述结构，打印相关信息。

#### Frida API的简单封装

就是更加简单使用hook

hookAddr

```
var soAddr = Module.findBaseAddress("libxiaojianbang.so");
hookAddr(soAddr.add(0x1ACC), 5); 	// Java_com_xiaojianbang_ndk_NativeHelper_add
hookAddr(soAddr.add(0x22A0), 3); 	// MD5Update

function hookAddr(funcAddr, paramsNum){
    var module = Process.findModuleByAddress(funcAddr);
    Interceptor.attach(funcAddr, {
        onEnter: function(args){
            this.logs = [];
            this.params = [];
            this.logs.push("call " + module.name + "!" + ptr(funcAddr).sub(module.base) + "\n");
            for(let i = 0; i < paramsNum; i++){
                this.params.push(args[i]);
                this.logs.push("this.args" + i + " onEnter: " + printAddr(args[i]));
            }
        }, onLeave: function(retval){
            for(let i = 0; i < paramsNum; i++){
                this.logs.push("this.args" + i + " onLeave: " + printAddr(this.params[i]));
            }
            this.logs.push("retval onLeave: " + printAddr(retval) + "\n");
            console.log(this.logs);
        }
    });
}
```

通过传入的函数地址，找到对应的模块，以便得到模块名和计算函数在模块中的偏移地址。对该函数地址进行Hook，在onEnter函数中输出一遍参数，在onLeave函数中输出一遍参数和返回值，以便处理C语言中常见的参数当返回值来使用的情况。将需要输出的信息，先添加到一个数组中，在onLeave函数执行完毕时，一次性输出，防止输出信息错乱。

#### 代码跟踪引擎stalker

Stalker是Frida的代码跟踪引擎，允许跟踪线程，捕获每个调用、每个块，甚至每个执行的指令。

实例  跟踪该函数内部的所有函数调用

```
var md5Addr = Module.getExportByName("libxiaojianbang.so", "Java_com_xiaojianbang_ndk_NativeHelper_md5");
Interceptor.attach(md5Addr, {
    onEnter: function () {
        this.tid = Process.getCurrentThreadId();
        Stalker.follow(this.tid, {
            events: {
                call: true,
            },
            onReceive(events) {
                var _events = Stalker.parse(events);
                for (var i = 0; i < _events.length; i++) {
                    console.log(_events[i]);
                }
            },
        });
    }, onLeave: function () {
        Stalker.unfollow(this.tid);
    }
});
/*
call,0x7590e77054,0x7590d63f60,0
call,0x7590e035a8,0x7590e0b24c,0
......
*/

```

在onEnter函数中，通过Stalker.follow跟踪函数调用。查看在源码中的声明：

![image-20240307171615142](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081411056.png)

Stalker.follow函数接收两个参数，第0个参数是threadId，用于指定跟踪的线程，默认是当前线程。在示例代码中，使用Process.getCurrentThreadId()来获取当前线程id，赋值给this.tid，并将该值传递给Stalker.unfollow，用来解除追踪。Stalker.follow函数的第1个参数是options，类型为StalkerOptions，用于自定义跟踪选项。在源码中的声明如下：

![image-20240307171631241](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081411057.png)

events用于指定生成哪些事件，传递给onReceive和onCallSummary。比如当events里面的call为true，Stalker在跟踪代码时，遇到函数调用，就会记录一些信息传递给onReceive和onCallSummary。events还支持在执行ret指令、所有指令exec、基本块block时生成事件。

这里就是可以更具自己需求更改

onReceive函数接收一个参数events，该参数可以使用Stalker.parse来解析。返回结果为StalkerEventFull数组或者StalkerEventBare数组。因此，示例代码中使用循环来遍历解析后的数组。Stalker.parse还可以指定StalkerParseOptions，查看在源码中的声明：

![image-20240307171700600](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081411058.png)

```
onReceive(events) {
	var _events = Stalker.parse(events);
	for (var i = 0; i < _events.length; i++) {
		var addr1 = _events[i][1];
		var module1 = Process.findModuleByAddress(addr1);
		if (module1 && module1.name == "libxiaojianbang.so") {
			var addr2 = _events[i][2];
			var module2 = Process.findModuleByAddress(addr2);
			console.log(module1.name, addr1.sub(module1.base), module2.name, addr2.sub(module2.base));
		}
	}
}
/*
libxiaojianbang.so 0x1f64 libxiaojianbang.so 0x1440
libxiaojianbang.so 0x1710 libxiaojianbang.so 0x1630
libxiaojianbang.so 0x187c libart.so 0x34f218
......
*/
```

Stalker.follow会跟踪所有so文件中的调用，包括一些系统so文件。上述代码中的addr1是发生call时的地址，如果只需要输出当前so文件中的调用，可以找到addr1对应的模块，判断一下即可。上述代码的第3行输出表示发生call的地址是在libxiaojianbang.so中的0x187c处，被调用的函数定义在libart.so中的0x34f218。