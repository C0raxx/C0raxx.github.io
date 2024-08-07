---
layout:     post
title:      【Android改机】Fart移植到Android10（整体脱壳）
subtitle:   Android改机
date:       2024-08-03
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【Android源码】
    - 【逆向】
    - 【Android】
---

# Fart移植到Android10（整体脱壳）

本篇主要是Fart整体脱壳函数的移植，其实在FART里，**整体和抽取都是一样拖得**，但是我学习的这个课程也就是小肩膀师傅的课程把这玩意分开来分析，我也这样做吧，感觉可以细致一点。

关于壳的基础知识，例如壳的发展，双亲委派机制，一代二代壳的原理主要在前面几篇博客里面已经做了一些分享和学习，这里我们就只做一些关于**Fart移植相关内容的学习，涉及到Fart如何进行整体脱壳，以及脱壳点选择的原因**。

省流资源分享

```
https://github.com/C0raxx/AndroidCompile/blob/main/Android10/Android10Fart_mini/DownLoadUrl
```



# 1.前置知识

```
dex的加载流程
通过mCookie脱壳的
通过openCommen函数脱壳的
通过DexFile构造函数脱壳的
youpk：通过ClassLinker的DexCacheData进一步得到DexFile

dex2oat的编译流程
通过修改dex2oat脱壳的

类的加载、链接、校验、初始化流程
DexHunter在defineClass进行类解析
LoadMethod、LinkCode

函数执行过程中的脱壳点
FART: Execute整体脱壳
FART：ArtMethod::invoke函数中进行dump CodeItem
youpk：直接到了解释执行的函数中进行dump CodeItem

```

源码查看网站

```
http://androidxref.com/
http://aospxref.com/
https://android-opengrok.bangnimang.net/
https://cs.android.com/ 
```

## 2.Execute修改

Execute基本上快到解释执行的位置了，Fart也是**选择这个函数进行一个脱壳**，它是这样的。

简单判断，然后调用dumpdexfilebyExecute

![image-20240807193833339](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712189.png)

## 3.Art_method.cc修改

```c++
extern "C" void dumpdexfilebyExecute(ArtMethod* artmethod)  REQUIRES_SHARED(Locks::mutator_lock_) {
			char *dexfilepath=(char*)malloc(sizeof(char)*1000);	
			if(dexfilepath==nullptr)
			{
				LOG(ERROR)<< "ArtMethod::dumpdexfilebyArtMethod,methodname:"<<artmethod->PrettyMethod().c_str()<<"malloc 1000 byte failed";
				return;
			}
			int result=0;
			int fcmdline =-1;
			char szCmdline[64]= {0};
			char szProcName[256] = {0};
			int procid = getpid();//进程pid
			sprintf(szCmdline,"/proc/%d/cmdline", procid);
			fcmdline = open(szCmdline, O_RDONLY,0644);//得到包名
			if(fcmdline >0)
			{
				result=read(fcmdline, szProcName,256);//得到了进程名字
				if(result<0)
				{
					LOG(ERROR) << "ArtMethod::dumpdexfilebyArtMethod,open cmdline file error";
					}
				close(fcmdline);
				
			}
			
			if(szProcName[0])
			{
				
					  const DexFile* dex_file = artmethod->GetDexFile(); //可以得到dexfile了
					  const uint8_t* begin_=dex_file->Begin();  // Start of data.
					  size_t size_=dex_file->Size();  // Length of data.
					  
					  memset(dexfilepath,0,1000);
					  int size_int_=(int)size_;
					  
					  memset(dexfilepath,0,1000);
					  sprintf(dexfilepath,"%s","/sdcard/fart");
					  mkdir(dexfilepath,0777);
							  
					  memset(dexfilepath,0,1000);
					  sprintf(dexfilepath,"/sdcard/fart/%s",szProcName);
					  mkdir(dexfilepath,0777);
							  	  
					  memset(dexfilepath,0,1000);
					  sprintf(dexfilepath,"/sdcard/fart/%s/%d_dexfile_execute.dex",szProcName,size_int_);
					  int dexfilefp=open(dexfilepath,O_RDONLY,0666);
					  if(dexfilefp>0){
						  close(dexfilefp);
						  dexfilefp=0;
						  
						  }else{
									  int fp=open(dexfilepath,O_CREAT|O_APPEND|O_RDWR,0666);
									  if(fp>0)
									  {
										  result=write(fp,(void*)begin_,size_);
										  if(result<0)
										  {
											  LOG(ERROR) << "ArtMethod::dumpdexfilebyArtMethod,open dexfilepath error";
											  }
										  fsync(fp); 
										  close(fp);  
										  memset(dexfilepath,0,1000);
										  sprintf(dexfilepath,"/sdcard/fart/%s/%d_classlist_execute.txt",szProcName,size_int_);
										  int classlistfile=open(dexfilepath,O_CREAT|O_APPEND|O_RDWR,0666);
											if(classlistfile>0)
											{
												for (size_t ii= 0; ii< dex_file->NumClassDefs(); ++ii) //NumClassDefs 关于类的结构体，从里面获得classdef，在得到ClassDescriptor 这个就是类名
												{
													const DexFile::ClassDef& class_def = dex_file->GetClassDef(ii);
													const char* descriptor = dex_file->GetClassDescriptor(class_def);
													result=write(classlistfile,(void*)descriptor,strlen(descriptor));
													if(result<0)
													{
														LOG(ERROR) << "ArtMethod::dumpdexfilebyArtMethod,write classlistfile file error";
														
														}
													const char* temp="\n";
													result=write(classlistfile,(void*)temp,1);
													if(result<0)
													{
														LOG(ERROR) << "ArtMethod::dumpdexfilebyArtMethod,write classlistfile file error";
														
														}
													}
												  fsync(classlistfile); 
												  close(classlistfile); 
												
												}
										  }


									  }

					
			}
			
			if(dexfilepath!=nullptr)
			{
				free(dexfilepath);
				dexfilepath=nullptr;
			}
		
}
```

可以看到fart的整体脱壳，也就是execute这个点的脱壳函数不算太长。

主要做了两件事

```
/sdcard/fart/%s/%d_dexfile_execute.dex的写入

/sdcard/fart/%s/%d_classlist_execute.txt的写入
```

这些都源于const DexFile* dex_file = artmethod->GetDexFile();这里的获得了Dexfile对象

脱壳很大程度上都是围绕这个做点文章。



在原代码里面 dex的写入比较简单，txt的写入涉及到了对dex对象的解析，主要是NumClassDefs得到有些啥类。

## 4.源码移植

### Execute的修改

前面的声明

![image-20240807195544708](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712190.png)

```
  extern "C" void saveDexFilebyExecute(ArtMethod* artmethod);
```

可以注意到我们的函数名改了，为了规避一些检测。

![image-20240807195626440](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712191.png)

这里不变，直接用原来的代码。

### art_method的修改

```c++
extern "C" void saveDexFilebyExecute(ArtMethod* artmethod)  REQUIRES_SHARED(Locks::mutator_lock_) {
			char *dexfilepath=(char*)malloc(sizeof(char)*1000);
			if(dexfilepath==nullptr)
			{
				LOG(ERROR)<< "ArtMethod::saveDexFilebyExecute,methodname:"<<artmethod->PrettyMethod().c_str()<<"malloc 1000 byte failed";
				return;
			}
			int result=0;
			int fcmdline =-1;
			char szCmdline[64]= {0};
			char szProcName[256] = {0};
			int procid = getpid();
			sprintf(szCmdline,"/proc/%d/cmdline", procid);
			fcmdline = open(szCmdline, O_RDONLY | O_CREAT,0644);
			if(fcmdline >0)
			{
				result=read(fcmdline, szProcName,256);
				if(result<0)
				{
					LOG(ERROR) << "ArtMethod::saveDexFilebyExecute,open cmdline file error";
				  close(fcmdline);
				  return;
					}
				close(fcmdline);

			}

			if(szProcName[0])
			{

					  const DexFile* dex_file = artmethod->GetDexFile();
					  const uint8_t* begin_=dex_file->Begin();  // Start of data.
					  size_t size_=dex_file->Size();  // Length of data
					  int size_int_=(int)size_;

			      memset(dexfilepath,0,1000);
					  sprintf(dexfilepath,"/data/data/%s/xiaojianbang",szProcName);
					  mkdir(dexfilepath,0777);


					  memset(dexfilepath,0,1000);
					  sprintf(dexfilepath,"/data/data/%s/xiaojianbang/%d_dexfile_execute.dex",szProcName,size_int_);

					  if(access(dexfilepath,F_OK)!=-1){



						  }else{
									  int fp=open(dexfilepath,O_CREAT|O_APPEND|O_RDWR,0666);
									  if(fp>0)
									  {
										  result=write(fp,(void*)begin_,size_);
										  if(result<0)
										  {
											  LOG(ERROR) << "ArtMethod::dumpdexfilebyArtMethod,open dexfilepath error";
											  }
										  fsync(fp);
										  close(fp);
										  memset(dexfilepath,0,1000);
										  sprintf(dexfilepath,"/data/data/%s/xiaojianbang/%d_classlist_execute.txt",szProcName,size_int_);
										  int classlistfile=open(dexfilepath,O_CREAT|O_APPEND|O_RDWR,0666);
											if(classlistfile>0)
											{
												for (size_t ii= 0; ii< dex_file->NumClassDefs(); ++ii)
												{
													const dex::ClassDef& class_def = dex_file->GetClassDef(ii);
													const char* descriptor = dex_file->GetClassDescriptor(class_def);
													result=write(classlistfile,(void*)descriptor,strlen(descriptor));
													if(result<0)
													{
														LOG(ERROR) << "ArtMethod::dumpdexfilebyArtMethod,write classlistfile file error";

														}
													const char* temp="\n";
													result=write(classlistfile,(void*)temp,1);
													if(result<0)
													{
														LOG(ERROR) << "ArtMethod::dumpdexfilebyArtMethod,write classlistfile file error";

														}
													}
												  fsync(classlistfile);
												  close(classlistfile);

												}
										  }


									  }


			}

			if(dexfilepath!=nullptr)
			{
				free(dexfilepath);
				dexfilepath=nullptr;
			}

}
```

修改点一

open的第二参数需要更改

![image-20240807195043272](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712192.png)

修改点二

路径从原来的sdcard/fart更改到如下目录，能**规避一些检测**

![image-20240807195223561](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712193.png)

修改点三

这里不再是原来的dexfilefp，改用access查看是否可以使用

![image-20240807195329029](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712194.png)

大概就是些地方

## 5.编译

直接编译即可

make -j8

![image-20240805000513018](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712195.png)

# 总结

Fart在整体脱壳的脱壳点选在了execute，在进入的时候就直接dump，dump函数写在interpreter.cc，照顾主要是根据传入的影子方法，得到dexfile对象，根据它进行dex的dump，和其NumClassDefs结构，把类的表写入txt。