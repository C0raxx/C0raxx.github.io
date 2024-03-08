---
layout:     post
title:      【Android学习】《Frida协议分析》五
subtitle:   工具学习
date:       2024-03-02
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【Android应用安全实战：Frida协议分析】
    - 【逆向】
    - 【Android】
---

## 第五章

本章介绍了一部分Frida在so层的API，熟练运用这些API是学习后续章节的前提。获取Module、枚举符号、Hook so函数等在so层的逆向中是最基础的操作，也是需要读者牢记在心的API。之后会综合这些API来使用，介绍一些更深入、更实用的内容。



Frida框架so层基本应用

本章主要介绍Frida在so层的API

因为Hook Frida的Java层代码和so层代码的环境配置是一样的，JavaScript代码的注入方式也是一样的。

### 获取Module

Module是**Frida中**比较常用的类 提供了很多与模块相关的操作，如枚举导出表、枚举导入表、枚举符号表、获取导出函数地址、获取模块基址等。

#### 通过模块名来获取Module

最简单的获取module常用代码

![image-20240307142427677](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081408368.png)

源码中的声明 可以看到两种寻找方式

![image-20240307142439681](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081408369.png)

#### 通过地址来获取Module

![image-20240307142600225](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081408370.png)

源码中的申明

![image-20240307142612115](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081408371.png)

通过地址获取Module的方法，传入NativePointerValue类型的内存地址，返回Module。传入的内存地址只需是模块内的任一地址即可。也就是说，当得到了一个函数地址，可以通过这两个方法来快速知道该函数是在哪一个so文件中定义的。

对于快速定位JNI函数注册在哪个so中很有用。

```
Process.findModuleByAddress(address);
Process.getModuleByAddress(address);

function findModuleByAddress(address: NativePointerValue): Module | null;
function getModuleByAddress(address: NativePointerValue): Module;

interface ObjectWrapper {
    handle: NativePointer;
}
type NativePointerValue = NativePointer | ObjectWrapper;
```

#### Process中的常用属性和方法

之前介绍了Process中用于获取模块的方法。Process在Frida中较为常用，这一小节将对Process中的常用属性和方法做出整体介绍。

```
declare namespace Process {
    const id: number;
    const arch: Architecture;
    const platform: Platform;
    const pageSize: number;
    const pointerSize: number;
    ......
    function getCurrentThreadId(): ThreadId;
    function findModuleByAddress(address: NativePointerValue): Module | null;
    function getModuleByAddress(address: NativePointerValue): Module;
    function findModuleByName(name: string): Module | null;
    function getModuleByName(name: string): Module;
    function enumerateModules(): Module[];
    function findRangeByAddress(address: NativePointerValue): RangeDetails | null;
    function getRangeByAddress(address: NativePointerValue): RangeDetails;
    function setExceptionHandler(callback: ExceptionHandlerCallback): void;
}

```

![image-20240307143711804](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081408372.png)

![image-20240307143719614](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081408373.png)

用用看

```

console.log("pid: ", Process.id);
console.log("arch: ", Process.arch);
console.log("platform: ", Process.platform);
console.log("pageSize: ", Process.pageSize);
console.log("pointerSize: ", Process.pointerSize);
console.log("CurrentThreadId: ", Process. getCurrentThreadId());
var soAddr = Process.findModuleByName("libxiaojianbang.so").base;
console.log("soAddr: ", soAddr);
var range = Process.findRangeByAddress(Process.findModuleByName("libxiaojianbang.so").base);
console.log("Range: ", JSON.stringify(range));
/*
pid:  13170
arch:  arm64
platform:  linux
pageSize:  4096
pointerSize:  8
CurrentThreadId:  13231
soAddr:  0x743a8e2000
Range:  {"base":"0x743a8e2000","size":20480,"protection":"r-x","file":{"path":"/data/app/com.xiaojianbang.app-Qj8kZpS2qmejJj88S35LnQ==/lib/arm64/libxiaojianbang.so","offset":0,"size":0}}
*/
```

### 枚举符号

Frida框架在so层代码中如何枚举符号，包括枚举模块的导入表、枚举模块的导出表和枚举模块的符号表，最后再给出Module中常用属性和方法。

#### 枚举模块的导入表

可以先得到对应的Module，再通过Module中的enumerateImports方法来**枚举**该Module中的导入表，进而得到对应的导入函数地址。

![image-20240307143858145](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081408374.png)

![image-20240307143906872](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081408375.png)

思路就是 如果要得到libxiaojianbang.so中的导入函数sprintf的内存地址，可以枚举so文件的导入表，遍历导入表中的函数，当函数名是sprintf时，记录函数地址即可。

```
enumerateImports(): ModuleImportDetails[];

var imports = Process.getModuleByName("libxiaojianbang.so").enumerateImports();
console.log(JSON.stringify(imports[0]));
//{"type":"function","name":"__cxa_atexit","module":"/apex/com.android.runtime/lib/bionic/libc.so","address":"0xedf050b9"}

var improts = Process.findModuleByName("libxiaojianbang.so").enumerateImports();
var sprintf_addr = null;
for(let i = 0; i < improts.length; i++){
    let _import = improts[i];
    if(_import.name.indexOf("sprintf") != -1){
        sprintf_addr = _import.address;
        break;
    }
}
console.log("sprintf_addr: ", sprintf_addr);
//sprintf_addr:  0x7bc0debaa0
```

#### 枚举模块的导出表

一般会有一些导出函数，如JNI静态注册的函数、需要导出给其他so文件使用的函数，以及JNI_OnLoad函数等。

ida中

![image-20240307144005979](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081408376.png)

如果要Hook这些函数，也要先得到这些函数的地址。

要得到地址的思路就是 在得到对应的Module后，通过Module中的enumerateExports方法来枚举该Module中的导出表，进而得到对应的导出函数地址。

![image-20240307144040361](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081408377.png)

![image-20240307144046175](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081408378.png)

如果要得到libxiaojianbang.so中的导出函数_Z8MD5FinalP7MD5_CTXPh的内存地址，

```
enumerateExports(): ModuleExportDetails[];

var exports = Process.getModuleByName("libxiaojianbang.so").enumerateExports();
console.log(JSON.stringify(exports[0]));
//{"type":"function","name":"JNI_OnLoad","address":"0xc68995f1"}

var exports = Process.findModuleByName("libxiaojianbang.so").enumerateExports();
var MD5Final_addr = null;
for(let i = 0; i < exports.length; i++){
    let _export = exports[i];
    if(_export.name.indexOf("_Z8MD5FinalP7MD5_CTXPh") != -1){
        MD5Final_addr = _export.address;
        break;
    }
}
console.log("MD5Final_addr: ", MD5Final_addr);
//MD5Final_addr:  0x7ad0beb988
```

#### 枚举模块的符号表

在得到对应的Module后，可以通过Module中的enumerateSymbols方法来枚举该Module中的符号表，进而得到出现在符号表中的函数地址。

![image-20240307144537007](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081408379.png)

返回ModuleSymbolDetails的数组，ModuleSymbolDetails中的属性与之前介绍的ModuleExportDetails相差无几



如果要得到libart.so中RegisterNatives的内存地址

这里选择不带checkjni的函数

```
var symbols = Process.getModuleByName("libart.so").enumerateSymbols();
var RegisterNatives_addr = null;
for (let i = 0; i < symbols.length; i++) {
    var symbol = symbols[i];
    if(symbol.name.indexOf("CheckJNI") == -1 && symbol.name.indexOf("RegisterNatives") != -1) {
        RegisterNatives_addr = symbol.address;
    }
}
console.log("RegisterNatives_addr: ", RegisterNatives_addr);
//RegisterNatives_addr:  0x7b3ebe9158

```



对于App应用程序本身的so文件，通常符号表会被删除，使用enumerateExports枚举导出表即可。



如果不知道某个系统函数来自于哪个so文件，可以使用Process.enumerateModules()枚举所有Module，再通过Module中的enumerateSymbols枚举模块中的符号表，与符号表中的函数名一一比对，具体实现代码如下

```
function findFuncInWitchSo(funcName) {
    var modules = Process.enumerateModules();
    for (let i = 0; i < modules.length; i++) {
        let module = modules[i];
        let _symbols = module.enumerateSymbols();
        for (let j = 0; j < _symbols.length; j++) {
            let _symbol = _symbols[i];
            if(_symbol.name == funcName){
                return module.name + " " + JSON.stringify(_symbol);
            }
        }
        let _exports = module.enumerateExports();
        for (let j = 0; j < _exports.length; j++) {
            let _export = _exports[j];
            if(_export.name == funcName){
                return module.name + " " + JSON.stringify(_export);
            }
        }
    }
    return null;
}
console.log(findFuncInWitchSo('strcat'));
//libc.so {"type":"function","name":"strcat","address":"0x7bc0e0322c"}
```

#### Module中的常用属性和方法

```
declare class Module {
    name: string;			//模块名
    base: NativePointer;	//模块基址
    size: number;			//模块大小
    path: string;			//模块所在路径
    enumerateImports(): ModuleImportDetails[];	//枚举导入表
    enumerateExports(): ModuleExportDetails[];	//枚举导出表
    enumerateSymbols(): ModuleSymbolDetails[];	//枚举符号表
    findExportByName(exportName: string): NativePointer | null;	//获取导出函数地址
    getExportByName(exportName: string): NativePointer;		//获取导出函数地址
    static load(name: string): Module;							//加载指定模块
    static findBaseAddress(name: string): NativePointer | null;		//获取模块基址
    static getBaseAddress(name: string): NativePointer;			//获取模块基址
    //获取导出函数地址
static findExportByName(moduleName: string | null, exportName: string): NativePointer | null;	
//获取导出函数地址
    static getExportByName(moduleName: string | null, exportName: string): NativePointer;
}
```

### Frida Hook so函数

这也是so层基本应用中最重要的内容，包括Hook导出函数、从给定地址获取内存数据、Hook任意函数、获取指针参数返回值和获取函数执行结果

#### Hook导出函数

**想要对so函数进行Hook，必须先得到函数的内存地址。**

Module的findExportByName和getExportByName都可以用来获取导出函数的内存地址，并且都有静态方法和实例方法两种。

静态方法可以直接使用类名.方法名的方式来访问，传入两个参数，第一个参数是string类型的模块名，第二个参数是string类型的导出函数名（以汇编界面显示的名字为准），返回NativePointer类型的函数地址。

实例方法可以先获取到Module对象，再通过对象.方法名的方式来访问，传入string类型的导出函数名即可，返回NativePointer类型的函数地址。

得到NativePointer类型的函数地址后，就可以使用Interceptor的attach函数来进行Hook，可以使用Interceptor的detachAll函数来解除Hook。

源码

![image-20240307145010501](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081408380.png)

```
Interceptor.detachAll()不需要传任何参数，Interceptor.attach需要传入函数地址和被Hook函数触发时执行的回调函数。此处以一个案例来说明该函数的用法。
```

用例代码

![image-20240307145044595](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081408381.png)

这玩意是native实现

![image-20240307145102882](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081408382.png)

```
可以看出add函数为native函数，对应的函数实现在libxiaojianbang.so中。将测试App应用程序，用zip压缩软件打开，在lib目录下又有4个目录：arm64-v8a、armeabi-v7a、x86、x86_64。这4个目录下都有libxiaojianbang.so，这些so文件功能一样，但使用的汇编代码不一样。arm64-v8a目录下是arm64的so文件，armeabi-v7a目录下是arm32的so文件。在不同的平台下，系统会自动选择对应文件夹下的so文件来使用，具体规则如下。
```

![image-20240307145131909](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081408383.png)

把so拖进ida

![image-20240307145149392](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081408384.png)

以Java_com_xiaojianbang_ndk_NativeHelper_add为例，

```
var funcAddr = Module.findExportByName("libxiaojianbang.so", "Java_com_xiaojianbang_ndk_NativeHelper_add");
Interceptor.attach(funcAddr, {
    onEnter: function (args) {
        console.log(args[0]);
        console.log(args[1]);
        console.log(args[2]);
        console.log(this.context.x3.toInt32());
        console.log(args[4].toUInt32());
    }, onLeave: function (retval) {
        console.log(retval.toInt32());
        console.log(this.context.x0);
        console.log("取x0寄存器的最后三个bit位", this.context.x0 & 0x7);
    }
});
//add函数触发以后的输出为
/*
0x7bc3bd66c0
0x7fda079fb4
0x5
6
7
18
0x12
取x0寄存器的最后三个bit位 2
*/
```

Interceptor通过inlineHook的方式拦截代码执行，会修改被Hook处的16个字节。当add函数执行时，会先执行onEnter函数中的代码，接着执行原函数，最后执行onLeave函数中的代码。

onEnter函数接收一个参数args（变量名可随意定义），类型为InvocationArguments，查看在源码中的声明：

![image-20240307145531663](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081408385.png)

Java层声明的native方法到了so层会额外增加两个参数。第0个参数是JNIEnv∗类型的可以调用里面的很多方法来完成C/C++与Java的交互。第1个参数是jclass/jobject，如果native方法是静态方法，这个参数就是jclass，代表native方法所在的类。如果native方法是实例方法，这个参数就是jobject，代表native方法所在的类实例化出来的对象。因此，上述输出的args[0]是JNIEnv∗，args[1]是jclass，后续三个参数，分别对应Java层native方法声明中的三个参数。

onLeave函数接收一个参数retval（变量名可随意定义），类型为InvocationReturnValue，查看在源码中的声明：

![image-20240307145606562](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081408386.png)

so文件可以在Android应用启动时就加载，也可以在后续需要使用时再加载。比如本书的测试App应用程序HookDemo.apk，当按下按钮时，NativeHelper类就会被加载，然后去执行该类下的静态代码块中的代码，此时libxiaojianbang.so才加载。Hook需要在so文件加载之后才能进行，很多新手会忽视这个问题，导致Hook失败。

#### 从给定地址查看内存数据

hexdump函数 指的是把内存某一时刻的内容导出成文件形式。

```
declare function hexdump(target: ArrayBuffer | NativePointerValue, options?: HexdumpOptions): string;
interface HexdumpOptions {
    offset?: number;	//从给定的target偏移一定字节数开始dump，默认为0
    length?: number;	//指定dump的字节数，注意需要十进制的数值，默认16*16
    header?: boolean;	//返回的string中是否包含标题，默认为true
    ansi?: boolean;	//返回的string是否带颜色，默认为false
}
```

测试

```
var soAddr = Module.findBaseAddress("libxiaojianbang.so");
var data = hexdump(soAddr, {length: 16, header: false});
console.log(data);
//  74c6c39000  7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00  .ELF............

var soAddr = Module.findBaseAddress("libxiaojianbang.so");
var data = hexdump(soAddr, {offset: 4, length: 16, header: false});
console.log(data);
//  74c6c39004  02 01 01 00 00 00 00 00 00 00 00 00  
```

#### Hook任意函数

so中hook任意函数只需要函数的内存地址 可以api获取 也可以自己就按

能够通过Frida的API来获取的函数地址必须是出现在导入表、导出表、符号表中的函数，也就是必须是有符号的函数。自己计算函数地址是更加通用的方式，可以适用于任意函数。

##### so文件基址的获取方式

可以通过Module的findBaseAddress和getBaseAddress方法来获取。查看在源码中的声明：

![image-20240307145841573](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081408387.png)

测试用例

```
declare class Module {
......
    static findBaseAddress(name: string): NativePointer | null;
static getBaseAddress(name: string): NativePointer;
}

var soAddr = Module.findBaseAddress("libxiaojianbang.so");
console.log(soAddr);
//Module.getBaseAddress("libxiaojianbang.so")
//soAddr:  0x7b2e6c0000
```

##### 函数地址相对so文件基址的偏移

简单来说就是这

![image-20240307145935194](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081408388.png)

##### 函数地址的计算

如果是thumb指令，函数地址计算方式为so文件基址+函数地址相对so文件基址的偏移+1。如果是arm指令，函数地址计算方式为so文件基址+函数地址相对so文件基址的偏移。

thumb指令和arm指令可以通过汇编指令对应的opcode字节数来区分，前者两个字节，后者4个字节。

当有了地址之后

```
var soAddr = Module.findBaseAddress("libxiaojianbang.so");
var sub_1A0C = soAddr.add(0x1ACC);
Interceptor.attach(sub_1ACC, {
    onEnter: function (args) {
        console.log("sub_1ACC onEnter args[0]: ", args[0]);
        console.log("sub_1ACC onEnter args[1]: ", args[1]);
        console.log("sub_1ACC onEnter args[2]: ", args[2]);
        console.log("sub_1ACC onEnter args[3]: ", args[3]);
        console.log("sub_1ACC onEnter args[4]: ", args[4]);
    }, onLeave: function (retval) {
        console.log("sub_1ACC onLeave retval: ", retval);
    }
});
//sub_1ACC onEnter args[0]:  0x7bc3bd66c0
//sub_1ACC onEnter args[1]:  0x7fda079fb4
//sub_1ACC onEnter args[2]:  0x5
//sub_1ACC onEnter args[3]:  0x6
//sub_1ACC onEnter args[4]:  0x7
//sub_1ACC onLeave retval:  0x12
```

#### 获取指针参数返回值

返回值定义成void，参数定义成指针，然后在函数执行过程中，改变传入的实参。也就是说，函数执行前传入的实参在函数调用后会被改变，修改成了函数执行的结果。对于这一类参数，需要在进入onEnter函数时，保存参数的内存地址。在进入onLeave函数时，再去读取参数对应内存地址中的内容。
很常见的情况。

伪代码

![image-20240307150306225](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081408389.png)

hook

```
var soAddr = Module.findBaseAddress("libxiaojianbang.so");
var MD5Final = soAddr.add(0x3A78);
Interceptor.attach(MD5Final, {
    onEnter: function (args) {
        this.args1 = args[1];
    }, onLeave: function (retval) {
        console.log(hexdump(this.args1));
    }
});
/*
7ffc689cc8  41 be f1 ce 7f dc 3e 42 c0 e5 d9 40 ad 74 ac 00  A.....>B...@.t..
//logcat中的输出结果
//CMD5 md5Result: 41bef1ce7fdc3e42c0e5d940ad74ac00
*/
```

#### Frida inlineHook获取函数执行结果

以Hook libxiaojianbang.so中偏移地址0x1AF4的指令为例，

以Hook libxiaojianbang.so中偏移地址0x1AF4的指令为例，该指令处于Java_com_xiaojianbang_ndk_NativeHelper_add函数中。

![image-20240307150451555](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081408390.png)

可以hook到里的参数

```
var hookAddr = Module.findBaseAddress("libxiaojianbang.so").add(0x1AF4);
Interceptor.attach(hookAddr, {
    onEnter: function (args) {
        console.log("onEnter x8: ", this.context.x8.toInt32());
        console.log("onEnter x9: ", this.context.x9.toInt32());
    }, onLeave: function (retval) {
        console.log("onLeave x0: ", this.context.x0.toInt32());
    }
});
/*
onEnter x8:  11
onEnter x9:  7
onLeave x0:  18
*/
```

onEnter在这条指令执行之前执行，onLeave在这条指令执行之后执行。但是在代码层面，可以看出inlineHook与之前Hook函数的方式没有区别。使用inlinehook时，推荐直接访问寄存器，不推荐使用args和retval。

### Frida修改函数参数与返回值

#### 修改函数参数与返回值

直接看一个用例

```
var soAddr = Module.findBaseAddress("libxiaojianbang.so");
var addFunc = soAddr.add(0x1ACC);
Interceptor.attach(addFunc, {
    onEnter: function (args) {
        args[2] = ptr(100);
//this.context.x2 = 100;
        console.log(args[2].toInt32());
    }, onLeave: function (retval) {
        console.log(retval.toInt32());
        retval.replace(100);
//this.context.x0 = 100;
    }
});
/*
args[2]:  100
retval:  113
//logcat中的输出为
//CADD addResult: 100
*/
```

对于数值参数的修改，如果直接使用数值赋值，args［2］=100，会有expected a pointer的错误提示。onEnter函数接收一个参数为args，类型为NativePointer的数组。类型不匹配，自然报错。**任何时候都要清楚地知道变量的类型，才能更好地应用。因此，把数值100传入ptr函数，构建出NativePointer后赋值给args［2］即可。**当然也可以使用this.context.x2=100的方式来修改，这是修改寄存器中的值，不需要构建NativePointer。

#### 修改字符串参数

修改数值参数和修改字符串参数本质上是一样的，都是用NativePointer类型的值去替换。

##### 修改参数指向的内存

```
function stringToBytes(str){
    return hexToBytes(stringToHex(str));
}
function stringToHex(str) {
    return str.split("").map(function(c) {
        return ("0" + c.charCodeAt(0).toString(16)).slice(-2);
    }).join("");
}
function hexToBytes(hex) {
    for (var bytes = [], c = 0; c < hex.length; c += 2)
        bytes.push(parseInt(hex.substr(c, 2), 16));
    return bytes;
}
var MD5Update = Module.findExportByName("libxiaojianbang.so", "_Z9MD5UpdateP7MD5_CTXPhj");
Interceptor.attach(MD5Update, {
    onEnter: function (args) {
        if(args[1].readCString() == "xiaojianbang"){
             let newStr = "xiaojian\0";
             args[1].writeByteArray(stringToBytes(newStr));
             console.log(hexdump(args[1]));
             args[2] = ptr(newStr.length - 1);
             console.log(args[2].toInt32());
        }
    }, onLeave: function (retval) {
    }
});
/*
7b2e35bf50  78 69 61 6f 6a 69 61 6e 00 61 6e 67 00 00 c0 41  xiaojian.ang...A
8
//logcat中的输出结果
//CMD5 md5Result: 66b0451b7a00d82790d4910a7a3a4162
```

用于从指定地址开始读取C语言字符串，返回JavaScript的string类型的字符串（可以调用JavaScript的string相关的方法）。该方法接收一个参数，用于指定读取的字节数，如果没有指定，则读取到C语言字符串结尾标志（字节0）为止。当第1个参数传入的明文数据为xiaojianbang时，才进行修改操作，防止误改后两次调用MD5Update的参数。

将参数指向的内存修改以后，加密的结果也发生了变化，因此修改是成功的。这种方式的缺点是修改了真实内存，其他函数访问这块内存也会有影响。

##### 将内存中已有的字符串赋值给参数

```
var MD5Update = Module.findExportByName("libxiaojianbang.so", "_Z9MD5UpdateP7MD5_CTXPhj");
var strAddr = Module.findBaseAddress("libxiaojianbang.so").add(0x3CFD);
Interceptor.attach(MD5Update, {
    onEnter: function (args) {
        if(args[1].readCString() == "xiaojianbang"){
            args[1] = strAddr;
            console.log(hexdump(args[1]));
            args[2] = ptr(strAddr.readCString().length);
            console.log(args[2].toInt32());
        }
    }, onLeave: function (retval) {
    }
});
/*
7ae6787cfd  63 6f 6d 2f 78 69 61 6f 6a 69 61 6e 62 61 6e 67  com/xiaojianbang
7ae6787d0d  2f 6e 64 6b 2f 4e 61 74 69 76 65 48 65 6c 70 65  /ndk/NativeHelpe
7ae6787d1d  72 00 65 6e 63 6f 64 65 00 28 29 4c 6a 61 76 61  r.encode.()Ljava
33
//logcat中的输出结果
//CMD5 md5Result: f6190c61b22ec8efe63fade2c47d8a49
```

以so文件基址+字符串在so文件中的偏移地址的方式计算出字符串的内存地址strAddr，注意这个地址任何时候都不需要加1。再将strAddr赋值给第1个参数，并修改第2个参数的字符串长度。从输出结果中可以看出，明文被修改成了com/xiaojianbang/ndk/NativeHelper，并且结果也发生了变化。这种方式修改的缺陷是指向内存中已有的字符串，灵活性不够，不一定能满足需求。

##### 修改MD5_CTX结构体中的buffer和count

```
//stringToBytes函数的定义，参考上一小节
var MD5Update = Module.findExportByName("libxiaojianbang.so", "_Z9MD5UpdateP7MD5_CTXPhj");
Interceptor.attach(MD5Update, {
    onEnter: function (args) {
        this.args0 = args[0];
        this.args1 = args[1];
    }, onLeave: function (retval) {
        if(this.args1.readCString() == "xiaojianbang"){
            let newStr = "jianbang";
            this.args0.add(24).writeByteArray(stringToBytes(newStr));
            console.log(hexdump(this.args0.writeInt(newStr.length * 8)));
        }
    }
});
/*
7fda079f08  40 00 00 00 00 00 00 00 01 23 45 67 89 ab cd ef  @........#Eg....
7fda079f18  fe dc ba 98 76 54 32 10 6a 69 61 6e 62 61 6e 67  ....vT2.jianbang
7fda079f28  62 61 6e 67 00 00 00 00 d0 a0 07 da 7f 00 00 00  bang............
7fda079f38  78 b2 2f 3e 7b 00 00 00 4c b2 2f 3e 7b 00 00 00  x./>{...L./>{...
7fda079f48  00 00 00 00 00 00 00 00 06 00 00 00 00 00 00 00  ................
7fda079f58  63 01 63 01 00 00 00 00 10 00 00 00 10 00 00 00  c.c.............
//logcat中的输出结果
//CMD5 md5Result: ea54ded1bd8a592dd826fb919687f13f
*/

在onEnter函数里记录第0个参数MD5_CTX的地址和第1个参数char∗的地址。当MD5 Update函数执行完毕后，在onLeave函数里判断第1个参数的值为xiaojianbang时，才进行MD5_CTX结构体的修改。该结构体前8个字节用于记录原始明文的bit长度。之后是16个字节的初始化魔数，同样采用小端字节序，再往后就是64个字节的buffer。因此，从MD5_CTX的地址处偏移24个字节后，使用writeByteArray写入明文数据，然后使用NativePointer里面的writeInt，修改MD5_CTX结构体中用来表示明文bit长度的数据。writeInt方法在源码中的声明如下，写入的数值为小端字节序，并且返回NativePointer本身。
```

##### 在内存中构建新的字符串

申请一块内存，在内存中写入字符串，然后把内存首地址赋值给参数。Memory类里面的allocUtf8String可以完成这些操作。查看在源码中的声明

![image-20240307151241141](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081408391.png)

```
var MD5Update = Module.findExportByName("libxiaojianbang.so", "_Z9MD5UpdateP7MD5_CTXPhj");
var newStr = "xiaojianbang&liruyi";
var newStrAddr = Memory.allocUtf8String(newStr);
Interceptor.attach(MD5Update, {
    onEnter: function (args) {
        if(args[1].readCString() == "xiaojianbang"){
            args[1] = newStrAddr;
            console.log(hexdump(args[1]));
            args[2] = ptr(newStr.length);
            console.log(args[2].toInt32());
        }
    }, onLeave: function (retval) {
    }
});
/*
7b34a80060  78 69 61 6f 6a 69 61 6e 62 61 6e 67 26 6c 69 72  xiaojianbang&lir
7b34a80070  75 79 69 00 00 00 00 00 23 00 00 00 00 00 00 00  uyi.....#.......
19
//logcat中的输出结果
//CMD5 md5Result: 8f1968f06a1e62bb3d83119352cc26cc
*/
```

