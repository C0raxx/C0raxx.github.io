---
layout:     post
title:      【Android学习】《Frida协议分析》八
subtitle:   工具学习
date:       2024-03-05
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【Android应用安全实战：Frida协议分析】
    - 【逆向】
    - 【Android】
---

## 第八章

在此基础上进行Frida和Python的交互

### Frida的Python库使用

#### Frida注入方式

我喜欢用的

```
import frida,sys

	process=frida.get_usb_device().attach("com.dodonew.online")
	script=process.create_script(jsCode)
	script.load()		#脚本加载
	sys.stdin.read()	#CMD不会退出，便于控制台观察
```

在进行包名注入之前，首先需要将调试的安卓系统工作机通过USB连接到计算机上，通过方法get_usb_device或者get_remote_device可以将Frida框架与工作机连接起来，再使用attach方法将要调试的安卓应用的包名写入即可完成附加，这里以第2章中使用的某嘟牛应用的注入为例。

看看修改后的代码

```
jsCode="""
	Java.perform(function (){
    console.log("start hooking...");
    var jsonRequest = Java.use("com.dodonew.online.http.JsonRequest");
    var requestUtil = Java.use("com.dodonew.online.http.RequestUtil");
    var utils = Java.use("com.dodonew.online.util.Utils");
    jsonRequest.paraMap.implementation = function (a) {
        console.log("jsonRequest.paraMap is called");
        return this.paraMap(a);
    }
    jsonRequest.addRequestMap.overload('java.util.Map', 'int').implementation = function 	(addMap, a) {
        console.log("jsonRequest.addRequestMap is called");
        return this.addRequestMap(addMap, a);
    }
    requestUtil.encodeDesMap.overload('java.lang.String', 'java.lang.String', 	'java.lang.String').implementation = function (data, desKey, desIV) {
        console.log("data: ", data, desKey, desIV);
        var encodeDesMap = this.encodeDesMap(data, desKey, desIV);
        console.log("encodeDesMap: ", encodeDesMap);
        return encodeDesMap;
    }
    utils.md5.implementation = function (a) {
        console.log("sign data: ", a);
        var md5 = this.md5(a);
        console.log("sign: ", md5);
        return md5;
    }
});
"""
```

#### spawn方式启动与连接非标准端口

用spawn方法将安卓应用的包名传入，会得到对应进程的PID，再使用PID注入的方式进行调用即可

### Frida与Python交互

要想在JavaScript中使用send方法，需要在Python代码中进行事件注册：

```
import frida,sys

	def Fun(message,data):
    	if message['type']=='send':
        	print("send:{}".format(message['payload']))
   		else:
        	print(message)

	device=frida.get_remote_device().attach(12260)
	script=device.create_script(jsCode)
	script.on('message',Fun)
	script.load()#脚本加载
	sys.stdin.read()#CMD不会退出，便于控制台观察
```

### Frida的RPC调用

Remote Procedure Call，直译过来便是远程过程调用

这玩意我没用过

RPC调用之所以会出现，直接的原因便是计算机无法通过本地调用的方式完成请求，比如A机器上的程序要调用B机器上的过程或者函数，它是一种高层的计算机网络技术，不同主机间的复杂细节对开发者来说是透明的，只需要像调用本地服务一样调用远程程序就可以了。如图8-1所示，RPC调用的关键其实只有两部分，第一部分是主动发起远程过程调用，第二部分是接收处理远程调用的结果。

![image-20240307172115570](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081412642.png)

我理解是 让繁复的运算交给终端 而不是调试端

某个应用的加密方法异常复杂，很难使用Python代码复现，可以在Hook代码中进行主动调用，这里拿某嘟牛的MD5加密为例：

```
function test(data){
    	var result="";
    	Java.perform(function (){
        	result=Java.use('com.dodonew.online.util.Utils').md5(data);
    	});
    	return result;
		};
	var result=test('123456');
	console.log("result:",result);
```

就可以在JavaScript代码中使用rpc.exports开放RPC调用接口，如开放上边的test函数供Python程序调用：

![image-20240307172226648](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202403081412643.png)

### 某嘟牛Frida算法转发

先来讲讲算法原地加密。这里讲解的算法原地加密和Frida的主动调用有着千丝万缕的关系，事实上，原地加密是将复杂的加密交给应用本身去完成，本质上就是Python程序将请求转发给了JavaScript代码，JavaScript去主动调用了应用内的过程。

```
function hookTest(username, passward){
var result;
Java.perform(function(){
var time = new Date().getTime();

var signData = 'equtype=ANDROID&loginImei=Android352689082129358&timeStamp=' + 
time + '&userPwd=' + passward + '&username=' + username + '&key=sdlkjsdljf0j2fsjk';

var Utils = Java.use('com.dodonew.online.util.Utils');
var sign = Utils.md5(signData).toUpperCase();
console.log('sign: ', sign);
    
var encryptData = '{"equtype":"ANDROID","loginImei":"Android352689082129358",
"sign":"'+sign +'","timeStamp":"'+ time +'","userPwd":"' + passward + '","username":"' 
+ username + '"}';

var RequestUtil = Java.use('com.dodonew.online.http.RequestUtil');
var Encrypt = RequestUtil.encodeDesMap(encryptData, '65102933', '32028092');
console.log('Encrypt: ', Encrypt);
result = Encrypt;
});
return result;

rpc.exports = {
   rpcfunc: hookTest
};
}
```

调用该函数时，传入用户名和密码会进行一系列的加密行为。首先会生成一个时间戳，之后主动调用String方法，创建了一段字符串signData，再通过MD5算法得到sign值。Encrypt则进行了一段DES加密，也通过主动调用得出了加密结果。这种主动调用的方式，不需要关注加密的方法实现和代码还原，实现较为简单。

要用Python程序复现登录请求，需要编写如下代码

```
import requests, json
	import frida

	# 调用frida脚本
	process = frida.get_remote_device().attach("com.dodonew.online")
	script = process.create_script(jsCode)
	print('[*] Running')
	script.load()
	cipherText = script.exports.rpcfunc('123456', 'a123456')

	url = 'http://api.dodovip.com/api/user/login'
	data = json.dumps({"Encrypt": cipherText})
	headers = {
    "content-type": "application/json; charset=utf-8",
    "User-Agent": "Dalvik/2.1.0 (Linux; U; Android 10; Pixel Build/QP1A.191005.007.A3)"
	}
	html = requests.post(url=url, data=data, headers=headers)
	print(html.text)
```

