---
layout:     post
title:      【Android改机】Fart移植到Android10（抽取脱壳+Frida增强）
subtitle:   Android改机
date:       2024-08-04
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【Android改机】
    - 【逆向】
    - 【Android】
---

# Fart移植到Android10（抽取脱壳+Frida增强）

这一篇主要研究学习的是，Fart脱壳机的源代码，它如何起到抽取脱壳作用，以及如何将之移植到Android10Aosp中，并且在成功之后，如何用Frida主动调用Fart中的Api，利用其对指定的类进行脱壳，还有脱下来的东西如何进行dex重构。文章开头处分享了我成功编译并成功刷机的一个刷机包，pixel3。

至于抽取壳，双亲委派，方法调用流程，在之前的壳专题的博客有写，这里不会赘述很多，只是简单在前置知识做个铺垫。

省流资源分享

```
https://github.com/C0raxx/AndroidCompile/blob/main/Android10/Android10Fart_Normal_Frida/DownLoadUrl
```



## 1.前置知识

这里我们主要简单讲一下，抽取壳如何起作用，实现形式，类加载器，双亲委派，类的加载与初始化，方法调用流程，以及解决思路。

## 1.1抽取加固

抽取加固本质：提取出dex中方法体的字节码，并在**方法运行时还原**。

在真实的运用当中一般有三种，

```
抽空方法体代码，运行方法后回填，运行完后不再抽取 --> 延时保存
抽空方法体代码，运行方法后回填，运行完后又抽取 --> FART、youpk主动调用
抽空方法体代码，将原有函数体替换为解密代码，运行时解密执行
```

其中对原有Dex的处理也可以看到

```
原有函数体数据空间置0，保留原有空间
对dex文件进行重构，不保留原有空间，在还原数据时，修改CodeItemOffest
```

## 1.2类加载器

安卓中常见的类加载器

```
BootClassLoader：单例模式，用来加载系统类
BaseDexClassLoader：是PathClassLoader、DexClassLoader、InMemoryDexClassLoader的父类，dex和类加载的主要逻辑都是在BaseDexClassLoader完成的
PathClassLoader： 是默认使用的类加载器，用于加载app自身的dex
DexClassLoader： 用于实现插件化、热修复、dex加固等
InMemoryDexClassLoader： 安卓8.0以后引入，用于内存加载dex
```

为什么要写类加载器呢？一个是为后面双亲委派做铺垫，一个是**Fart解决思路需要获得所有的ClassLoader做主动调用**，下面是为啥要获得所有ClassLoader

```
所有函数 -- 所有类 -- 所有dex -- 所有ClassLoader
```

## 1.3双亲委派

简单来讲就是先找父ClassLoader加载，不行就一直向上找父ClassLoader，到了BootClassloader都不行就往下，找他所有的子加载器开始加载。

```
如果一个类加载器收到了类加载请求，会先把这个请求委托给父类的加载器去执行
如果父类加载器还存在其父类加载器，则进一步向上委托，依次类推，最终到达顶层的启动类加载器
如果父类加载器可以完成类加载任务，就成功返回，倘若父类加载器无法完成此加载任务，子加载器才会尝试自己去加载
```

为什么要讲双亲委派呢？一个是因为Fart构造主动调用链需要这个知识做铺垫，另外一个是后续某些加固会对类加载器进行一个修正，导致有些时候找不到我们的DexClassLoader，**从而无法主动调用壳加载的dex**。

```
插入ClassLoader
	BootClassLoader
	DexClassLoader
	PathClassLoader  mClassLoader
替换ClassLoader
	BootClassLoader
	PathClassLoader
	DexClassLoader mClassLoader
```



## 1.4类加载与初始化

Loadclass->findclass->pathlist.findclass->element.findclass->loadclassbinaryname->defineclass->defineclassnative->DexFile_defineclassnative->获取ClassLink，填入dex_cache（youpk脱壳点）初始化->defineClass(dexhunter脱壳点)->setupclass,loadclass,linkclass->(loadclass)里面loadfield，loadmethod->（loadmethod）**里面设置了codeitem offset**，dex方法索引，**Artmethod方法转化**，解释模式或者快速模式的执行->(linkclass)加载完成，开始初始化->InitializeClass判断，校验，初始化静态字段，clinit的调用

## 1.5方法调用流程

我们后续的fart源码当中，因为涉及主动调用，所以这一块也得看看。

invoke->Method_inovke->InvokeMethod等价转为Artmethod->InvokeMethodImpl->InvokeWithArray(jni调用也会走到这里)->Invoke（这里就是我们脱抽取壳的脱壳点）->如果是解释模式，**还是会走到Execute**

其中**类的初始化函数必走在解释模式**。

要注意这里的Invoke的脱壳时机是比Execute晚的，虽然调用链中Execute比较晚，但按照Fart当中的写法，也就是上一篇整体加固脱壳的思路，是写在cinit初始化函数中进行dex的dump的，而初始化函数又相较于其他的函数时比较早的，所以Fart当中的Invoke时机其实更加晚一点。

## 1.6解决思路

核心是：主动调用app中的所有函数，在函数执行过程中，**将方法体的CodeItem保存下来**。

tip：其实被动调用也可以 dexhunter脱壳机就是这样

```
被动调用
app正常运行过程中所发生的函数调用
只对dex中部分的类完成加载，只对dex中的部分函数完成调用
调用函数不全，导致能够恢复的函数有限

主动调用
构造虚拟调用，对app中所有函数完成调用
在这些函数执行时，保存函数体CodeItem数据
保存数据的时机越晚，效果越好
```

那么如何调用app中的所有函数呢？所有函数 -- 所有类 -- 所有dex -- 所有ClassLoader，这是正常的一个逻辑关系，而我们就要从最后的ClassLoader入手，**利用双亲委派关系，得到所有的ClassLoader**。

如何获取所有的ClassLoader
先得到一个ClassLoader，再通过ClassLoader的parent属性**向上遍历**。

如何先得到一个ClassLoader
PathClassLoader加载dex以后，会记录在LoadedApk的mClassLoader属性中默认使用这个ClassLoader去寻找类。

```
普通app运行流程
BootClassLoader加载系统核心库
PathClassLoader加载app自身dex  mClassLoader

加固app运行流程
BootClassLoader加载系统核心库
PathClassLoader加载壳的dex  mClassLoader
壳的dex/so加载原先app自身dex
```

而之前也讲过有些加固会**对类加载器进行修正，这也造成有的加固需要枚举所有ClassLoader**，才能找到app自身dex里的类，这我们后面结合Frida强化功能的时候再说。

## 2.Fart源码研究

这里只看一下重要功能的实现，包的导入，函数声明啥的暂且不表，后面移植的时候会写。

在**performLaunchActivity**中

![image-20240808053606390](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712321.png)

如下，可以看到核心功能来到fart

![image-20240808053644262](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712322.png)

fart如下，思路就是上面讲的通过**循环获得父classloader**，然后把每一轮循环的classloader作为参数给fartwithClassloader

![image-20240808053740189](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712323.png)

如下 比较长**，在里面获得这个类加载器加载的dex，然后通过这个dex里面的pathlist获得所有类，在获得所有函数，放置在ElementsArray当中，这也和我们在之前看类加载与初始化契合了**。重要的就是最后的loadClassAndInvoke，第三个参数是我们等会去native层看的函数（这个是fart添加上去的）

```
public static void fartwithClassloader(ClassLoader appClassloader) {
        List<Object> dexFilesArray = new ArrayList<Object>();
        Field pathList_Field = (Field) getClassField(appClassloader, "dalvik.system.BaseDexClassLoader", "pathList");
        Object pathList_object = getFieldOjbect("dalvik.system.BaseDexClassLoader", appClassloader, "pathList");
        Object[] ElementsArray = (Object[]) getFieldOjbect("dalvik.system.DexPathList", pathList_object, "dexElements");
        Field dexFile_fileField = null;
        try {
            dexFile_fileField = (Field) getClassField(appClassloader, "dalvik.system.DexPathList$Element", "dexFile");
        } catch (Exception e) {
                e.printStackTrace();
            } catch (Error e) {
                e.printStackTrace();
            }
        Class DexFileClazz = null;
        try {
            DexFileClazz = appClassloader.loadClass("dalvik.system.DexFile");
        } catch (Exception e) {
                e.printStackTrace();
            } catch (Error e) {
                e.printStackTrace();
            }
        Method getClassNameList_method = null;
        Method defineClass_method = null;
        Method dumpDexFile_method = null;
        Method dumpMethodCode_method = null;

        for (Method field : DexFileClazz.getDeclaredMethods()) {
            if (field.getName().equals("getClassNameList")) {
                getClassNameList_method = field;
                getClassNameList_method.setAccessible(true);
            }
            if (field.getName().equals("defineClassNative")) {
                defineClass_method = field;
                defineClass_method.setAccessible(true);
            }
            if (field.getName().equals("dumpDexFile")) {
                dumpDexFile_method = field;
                dumpDexFile_method.setAccessible(true);
            }
            if (field.getName().equals("dumpMethodCode")) {
                dumpMethodCode_method = field;
                dumpMethodCode_method.setAccessible(true);
            }
        }
        Field mCookiefield = getClassField(appClassloader, "dalvik.system.DexFile", "mCookie");
        Log.v("ActivityThread->methods", "dalvik.system.DexPathList.ElementsArray.length:" + ElementsArray.length);//5个
        for (int j = 0; j < ElementsArray.length; j++) {
            Object element = ElementsArray[j];
            Object dexfile = null;
            try {
                dexfile = (Object) dexFile_fileField.get(element);
            } catch (Exception e) {
                e.printStackTrace();
            } catch (Error e) {
                e.printStackTrace();
            }
            if (dexfile == null) {
                Log.e("ActivityThread", "dexfile is null");
                continue;
            }
            if (dexfile != null) {
                dexFilesArray.add(dexfile);
                Object mcookie = getClassFieldObject(appClassloader, "dalvik.system.DexFile", dexfile, "mCookie");
                if (mcookie == null) {
                    Object mInternalCookie = getClassFieldObject(appClassloader, "dalvik.system.DexFile", dexfile, "mInternalCookie");
                    if(mInternalCookie!=null)
                    {
						mcookie=mInternalCookie;
					}else{
							Log.v("ActivityThread->err", "get mInternalCookie is null");
							continue;
							}
                    
                }
                String[] classnames = null;
                try {
                    classnames = (String[]) getClassNameList_method.invoke(dexfile, mcookie);
                } catch (Exception e) {
                    e.printStackTrace();
                    continue;
                } catch (Error e) {
                    e.printStackTrace();
                    continue;
                }
                if (classnames != null) {
                    for (String eachclassname : classnames) {
                        loadClassAndInvoke(appClassloader, eachclassname, dumpMethodCode_method);
                    }
                }

            }
        }
        return;
    }
```

如下，重点只关注dumpMethodCode_method.invoke(null, constructor);这个dumpMethodCode_method也就是我们上面传进的dumpMethodCode_method，调用里面的invoke方法。好我们继续跟。

```
public static void loadClassAndInvoke(ClassLoader appClassloader, String eachclassname, Method dumpMethodCode_method) {
        Class resultclass = null;
        Log.i("ActivityThread", "go into loadClassAndInvoke->" + "classname:" + eachclassname);
        try {
            resultclass = appClassloader.loadClass(eachclassname);
        } catch (Exception e) {
            e.printStackTrace();
            return;
        } catch (Error e) {
            e.printStackTrace();
            return;
        }
        if (resultclass != null) {
            try {
                Constructor<?> cons[] = resultclass.getDeclaredConstructors();
                for (Constructor<?> constructor : cons) {
                    if (dumpMethodCode_method != null) {
                        try {
                            dumpMethodCode_method.invoke(null, constructor);
                        } catch (Exception e) {
                            e.printStackTrace();
                            continue;
                        } catch (Error e) {
                            e.printStackTrace();
                            continue;
                        }
                    } else {
                        Log.e("ActivityThread", "dumpMethodCode_method is null ");
                    }

                }
            } catch (Exception e) {
                e.printStackTrace();
            } catch (Error e) {
                e.printStackTrace();
            }
            try {
                Method[] methods = resultclass.getDeclaredMethods();
                if (methods != null) {
                    for (Method m : methods) {
                        if (dumpMethodCode_method != null) {
                            try {
                               dumpMethodCode_method.invoke(null, m);
                             } catch (Exception e) {
                                e.printStackTrace();
                                continue;
                            } catch (Error e) {
                                e.printStackTrace();
                                continue;
                            }
                        } else {
                            Log.e("ActivityThread", "dumpMethodCode_method is null ");
                        }
                    }
                }
            } catch (Exception e) {
                e.printStackTrace();
            } catch (Error e) {
                e.printStackTrace();
            }
        }
    }
```

来到这里，先转为Art方法，然后myfartInvoke调用

![image-20240808055129266](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712324.png)

里面对参数进行了一些处理，可以看到这样的参数是**不可能正常地调用**的，也就是nullptr等会会用做是不是farinvoke传进来地检测

![image-20240808055207498](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712325.png)

来到这里，可以看到我们修改了在这里invoke进去的时候就修改了源码，添加了dumpArtMethod这个方法，我们看看这个东西是啥

![image-20240808054551969](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712326.png)

这就是最核心的地方了，前后涉及三个文件的操作，分别是dex，txt，bin。后续需要用dex，bin来重构dex

有了**ArtMethod就很容易获得dexfile对象，这是个结构体**，里面有dex的起始和大小，直接写出了。

然后txt的写出，这里还是跟之前一样的思路**，获取NumClassDefs，作遍历**，得到所有的dexfile，然后获取所有类，形成一个list。

最后的也是最重要的，就是codeitem的写出，这一块首先是**计算整个codeite的长度**（源码里面涉及了trysize的计算，我们自己的移植的时候有别的办法）然后**以json的格式写入codeitem的bin**，codeitem部分base64编码。

```
extern "C" void dumpArtMethod(ArtMethod* artmethod)  REQUIRES_SHARED(Locks::mutator_lock_) {
			char *dexfilepath=(char*)malloc(sizeof(char)*1000);	
			if(dexfilepath==nullptr)
			{
				LOG(ERROR) << "ArtMethod::dumpArtMethodinvoked,methodname:"<<artmethod->PrettyMethod().c_str()<<"malloc 1000 byte failed";
				return;
			}
			int result=0;
			int fcmdline =-1;
			char szCmdline[64]= {0};
			char szProcName[256] = {0};
			int procid = getpid();
			sprintf(szCmdline,"/proc/%d/cmdline", procid);
			fcmdline = open(szCmdline, O_RDONLY,0644);
			if(fcmdline >0)
			{
				result=read(fcmdline, szProcName,256);
				if(result<0)
				{
					LOG(ERROR) << "ArtMethod::dumpdexfilebyArtMethod,open cmdline file file error";									
				}
				close(fcmdline);
			}
			
			if(szProcName[0])
			{
					  const DexFile* dex_file = artmethod->GetDexFile();
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
					  sprintf(dexfilepath,"/sdcard/fart/%s/%d_dexfile.dex",szProcName,size_int_);
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
												LOG(ERROR) << "ArtMethod::dumpdexfilebyArtMethod,open dexfilepath file error";
														
											}
										  fsync(fp); 
										  close(fp);  
										  memset(dexfilepath,0,1000);
										  sprintf(dexfilepath,"/sdcard/fart/%s/%d_classlist.txt",szProcName,size_int_);
										  int classlistfile=open(dexfilepath,O_CREAT|O_APPEND|O_RDWR,0666);
											if(classlistfile>0)
											{
												for (size_t ii= 0; ii< dex_file->NumClassDefs(); ++ii) 
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
						  const DexFile::CodeItem* code_item = artmethod->GetCodeItem();
						  if (LIKELY(code_item != nullptr)) 
						  {
							  
					  
								  int code_item_len = 0;
								  uint8_t *item=(uint8_t *) code_item;
								  if (code_item->tries_size_>0) {
									  const uint8_t *handler_data = (const uint8_t *)(DexFile::GetTryItems(*code_item, code_item->tries_size_));
									  uint8_t * tail = codeitem_end(&handler_data);
									  code_item_len = (int)(tail - item);
								  }else{
									  code_item_len = 16+code_item->insns_size_in_code_units_*2;
								  }  
									  memset(dexfilepath,0,1000);
									  int size_int=(int)dex_file->Size();  
									  uint32_t method_idx=artmethod->GetDexMethodIndexUnchecked();
									  sprintf(dexfilepath,"/sdcard/fart/%s/%d_ins_%d.bin",szProcName,size_int,(int)gettidv1());
								      int fp2=open(dexfilepath,O_CREAT|O_APPEND|O_RDWR,0666);
									  if(fp2>0){
										  lseek(fp2,0,SEEK_END);
										  memset(dexfilepath,0,1000);
										  int offset=(int)(item - begin_);
										  sprintf(dexfilepath,"{name:%s,method_idx:%d,offset:%d,code_item_len:%d,ins:",artmethod->PrettyMethod().c_str(),method_idx,offset,code_item_len);
										  int contentlength=0;
										  while(dexfilepath[contentlength]!=0) contentlength++;
										  result=write(fp2,(void*)dexfilepath,contentlength);
										  if(result<0)
													{
														LOG(ERROR) << "ArtMethod::dumpdexfilebyArtMethod,write ins file error";
														
														}
										  long outlen=0;
										  char* base64result=base64_encode((char*)item,(long)code_item_len,&outlen);
										  result=write(fp2,base64result,outlen);
										  if(result<0)
													{
														LOG(ERROR) << "ArtMethod::dumpdexfilebyArtMethod,write ins file error";
														
														}
										  result=write(fp2,"};",2);
										  if(result<0)
													{
														LOG(ERROR) << "ArtMethod::dumpdexfilebyArtMethod,write ins file error";
														
														}
										  fsync(fp2); 
										  close(fp2);
										  if(base64result!=nullptr){
											  free(base64result);
											  base64result=nullptr;
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

OK，到这里就差不多了。

## 3.源码移植

入口还是一样，只不过我们不打算把fartthread这些函数的实现逻辑写在ActivityThread里面，我们只需要在handlebindapplication里面调用这个fartthread线程这ok。**移植的时候尽可能抹去其特征**。

我选择在这

![image-20240808061107950](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712327.png)

然后创建一个新的包

![image-20240808061133360](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712328.png)

```
package com.corax;
import android.app.Application;
import android.util.Log;

import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;

public class savedex {
    //add
    public static Field getClassField(ClassLoader classloader, String class_name,
                                      String filedName) {

        try {
            Class obj_class = classloader.loadClass(class_name);//Class.forName(class_name);
            Field field = obj_class.getDeclaredField(filedName);
            field.setAccessible(true);
            return field;
        } catch (SecurityException e) {
            e.printStackTrace();
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalArgumentException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        return null;

    }

    public static Object getClassFieldObject(ClassLoader classloader, String class_name, Object obj,
                                             String filedName) {

        try {
            Class obj_class = classloader.loadClass(class_name);//Class.forName(class_name);
            Field field = obj_class.getDeclaredField(filedName);
            field.setAccessible(true);
            Object result = null;
            result = field.get(obj);
            return result;
        } catch (SecurityException e) {
            e.printStackTrace();
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalArgumentException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
        return null;

    }

    public static Object invokeStaticMethod(String class_name,
                                            String method_name, Class[] pareTyple, Object[] pareVaules) {

        try {
            Class obj_class = Class.forName(class_name);
            Method method = obj_class.getMethod(method_name, pareTyple);
            return method.invoke(null, pareVaules);
        } catch (SecurityException e) {
            e.printStackTrace();
        } catch (IllegalArgumentException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        return null;

    }

    public static Object getFieldOjbect(String class_name, Object obj,
                                        String filedName) {
        try {
            Class obj_class = Class.forName(class_name);
            Field field = obj_class.getDeclaredField(filedName);
            field.setAccessible(true);
            return field.get(obj);
        } catch (SecurityException e) {
            e.printStackTrace();
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalArgumentException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (NullPointerException e) {
            e.printStackTrace();
        }
        return null;

    }


    public static void loadClassAndInvoke(ClassLoader appClassloader, String eachclassname, Method _saveMethodCode) {
        Class resultclass = null;
        Log.i("corax", "go into loadClassAndInvoke->" + "classname:" + eachclassname);
        try {
            resultclass = appClassloader.loadClass(eachclassname);
        } catch (Exception e) {
            e.printStackTrace();
            return;
        } catch (Error e) {
            e.printStackTrace();
            return;
        }
        if (resultclass != null) {
            try {
                Constructor<?> cons[] = resultclass.getDeclaredConstructors();
                for (Constructor<?> constructor : cons) {
                    if (_saveMethodCode != null) {
                        try {
                            _saveMethodCode.invoke(null, constructor);
                        } catch (Exception e) {
                            e.printStackTrace();
                            continue;
                        } catch (Error e) {
                            e.printStackTrace();
                            continue;
                        }
                    } else {
                        Log.e("corax", "dumpMethodCode_method is null ");
                    }

                }
            } catch (Exception e) {
                e.printStackTrace();
            } catch (Error e) {
                e.printStackTrace();
            }
            try {
                Method[] methods = resultclass.getDeclaredMethods();
                if (methods != null) {
                    for (Method m : methods) {
                        if (_saveMethodCode != null) {
                            try {
                                _saveMethodCode.invoke(null, m);
                            } catch (Exception e) {
                                e.printStackTrace();
                                continue;
                            } catch (Error e) {
                                e.printStackTrace();
                                continue;
                            }
                        } else {
                            Log.e("corax", "saveMethodCode_method is null ");
                        }
                    }
                }
            } catch (Exception e) {
                e.printStackTrace();
            } catch (Error e) {
                e.printStackTrace();
            }
        }
    }

    public static void coraxwithClassloader(ClassLoader appClassloader) {
        List<Object> dexFilesArray = new ArrayList<Object>();
        Object pathList_object = getFieldOjbect("dalvik.system.BaseDexClassLoader", appClassloader, "pathList");
        Object[] ElementsArray = (Object[]) getFieldOjbect("dalvik.system.DexPathList", pathList_object, "dexElements");
        Field dexFile_fileField = null;
        try {
            dexFile_fileField = (Field) getClassField(appClassloader, "dalvik.system.DexPathList$Element", "dexFile");
        } catch (Exception e) {
            e.printStackTrace();
        } catch (Error e) {
            e.printStackTrace();
        }
        Class DexFileClazz = null;
        try {
            DexFileClazz = appClassloader.loadClass("dalvik.system.DexFile");
        } catch (Exception e) {
            e.printStackTrace();
        } catch (Error e) {
            e.printStackTrace();
        }
        Method _getClassNameList = null;
        Method _saveMethodCode = null;

        for (Method field : DexFileClazz.getDeclaredMethods()) {
            if (field.getName().equals("getClassNameList")) {
                _getClassNameList = field;
                _getClassNameList.setAccessible(true);
            }
            if (field.getName().equals("saveMethodCode")) {
                _saveMethodCode = field;
                _saveMethodCode.setAccessible(true);
            }
        }
        Log.v("ActivityThread->methods", "dalvik.system.DexPathList.ElementsArray.length:" + ElementsArray.length);//5个
        for (int j = 0; j < ElementsArray.length; j++) {
            Object element = ElementsArray[j];
            Object dexfile = null;
            try {
                dexfile = (Object) dexFile_fileField.get(element);
            } catch (Exception e) {
                e.printStackTrace();
            } catch (Error e) {
                e.printStackTrace();
            }
            if (dexfile == null) {
                Log.e("ActivityThread", "dexfile is null");
                continue;
            }
            if (dexfile != null) {
                dexFilesArray.add(dexfile);
                Object mcookie = getClassFieldObject(appClassloader, "dalvik.system.DexFile", dexfile, "mCookie");
                if (mcookie == null) {
                    Object mInternalCookie = getClassFieldObject(appClassloader, "dalvik.system.DexFile", dexfile, "mInternalCookie");
                    if(mInternalCookie!=null)
                    {
                        mcookie=mInternalCookie;
                    }else{
                        Log.v("ActivityThread->err", "get mInternalCookie is null");
                        continue;
                    }

                }
                String[] classnames = null;
                try {
                    classnames = (String[]) _getClassNameList.invoke(dexfile, mcookie);
                } catch (Exception e) {
                    e.printStackTrace();
                    continue;
                } catch (Error e) {
                    e.printStackTrace();
                    continue;
                }
                if (classnames != null) {
                    for (String eachclassname : classnames) {
                        loadClassAndInvoke(appClassloader, eachclassname, _saveMethodCode);
                    }
                }

            }
        }
    }

    public static ClassLoader getClassloader() {
        ClassLoader resultClassloader = null;
        Object currentActivityThread = invokeStaticMethod(
                "android.app.ActivityThread", "currentActivityThread",
                new Class[]{}, new Object[]{});
        Object mBoundApplication = getFieldOjbect(
                "android.app.ActivityThread", currentActivityThread,
                "mBoundApplication");
        Object loadedApkInfo = getFieldOjbect(
                "android.app.ActivityThread$AppBindData",
                mBoundApplication, "info");
        Application mApplication = (Application) getFieldOjbect("android.app.LoadedApk", loadedApkInfo, "mApplication");
        resultClassloader = mApplication.getClassLoader();
        return resultClassloader;
    }

    public static void corax() {
        ClassLoader appClassloader = getClassloader();
        ClassLoader parentClassloader=appClassloader.getParent();
        if(!appClassloader.toString().contains("java.lang.BootClassLoader"))
        {
            coraxwithClassloader(appClassloader);
        }
        while(parentClassloader!=null){
            if(!parentClassloader.toString().contains("java.lang.BootClassLoader"))
            {
                coraxwithClassloader(parentClassloader);
            }
            parentClassloader=parentClassloader.getParent();
        }
    }
    public static void coraxthread() {
        new Thread(new Runnable() {

            @Override
            public void run() {
                // TODO Auto-generated method stub
                try {
                    Log.e("corax", "start sleep......");
                    Thread.sleep(1 * 60 * 1000);
                } catch (InterruptedException e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                }
                Log.e("corax", "sleep over and start run");
                corax();
                Log.e("corax", " run over");

            }
        }).start();
    }
    //add
}
```

上面的代码起始改的不是很多，主要集中于函数名的修改。直接用我这个就可以了。

记得在ActivityThread.java当中把包导进去

![image-20240808061343360](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712329.png)

OK，java层面到此结束。

native层的saveMethodCode需要注册，在这

![image-20240808061520131](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712330.png)

上面这样写，里面调用另外两个native函数

![image-20240808061546350](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712331.png)

其中mycoraxinvoke就在art_method.cc里面

![image-20240808061751486](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712332.png)

另外的一个在

![image-20240808061858137](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712333.png)

两个函数定义好了，要在原来的这里 声明

![image-20240808061931877](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712334.png)

然后是invoke的添加

![image-20240808062053154](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712335.png)

按照之前源码研究，基本上也来到最后了saveArtMethod了

![image-20240808062135380](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712336.png)

```
extern "C" void saveArtMethod(ArtMethod* artmethod)  REQUIRES_SHARED(Locks::mutator_lock_) {
			char *dexfilepath=(char*)malloc(sizeof(char)*1000);
			if(dexfilepath==nullptr)
			{
				LOG(ERROR) << "ArtMethod::dumpArtMethodinvoked,methodname:"<<artmethod->PrettyMethod().c_str()<<"malloc 1000 byte failed";
				return;
			}
			int result=0;
			int fcmdline =-1;
			char szCmdline[64]= {0};
			char szProcName[256] = {0};
			int procid = getpid();
			sprintf(szCmdline,"/proc/%d/cmdline", procid);
			fcmdline = open(szCmdline, O_RDONLY | O_CREAT ,0644);
			if(fcmdline >0)
			{
				result=read(fcmdline, szProcName,256);
				if(result<0)
				{
					LOG(ERROR) << "ArtMethod::dumpdexfilebyArtMethod,open cmdline file file error";
				    close(fcmdline);
				    return;
				}
				close(fcmdline);
			}

			if(szProcName[0])
			{
					  const DexFile* dex_file = artmethod->GetDexFile();
					  const uint8_t* begin_=dex_file->Begin();  // Start of data.
					  size_t size_=dex_file->Size();  // Length of data.
					  int size_int_=(int)size_;

					  memset(dexfilepath,0,1000);
					  sprintf(dexfilepath,"/data/data/%s/corax",szProcName);
					  mkdir(dexfilepath,0777);

					  memset(dexfilepath,0,1000);
					  sprintf(dexfilepath,"/data/data/%s/corax/%d_artmethod.dex",szProcName,size_int_);
					  if(access(dexfilepath,F_OK) !=-1 ){
						  }else{
									  int fp=open(dexfilepath,O_CREAT|O_APPEND|O_RDWR,0666);
									  if(fp>0)
									  {
										  result=write(fp,(void*)begin_,size_);
										  if(result<0)
											{
												LOG(ERROR) << "ArtMethod::dumpdexfilebyArtMethod,open dexfilepath file error";

											}
										  fsync(fp);
										  close(fp);
										  memset(dexfilepath,0,1000);
										  sprintf(dexfilepath,"/data/data/%s/corax/%d_classlist_artmethod.txt",szProcName,size_int_);
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
						  const dex::CodeItem* code_item = artmethod->GetCodeItem();
						  if (LIKELY(code_item != nullptr))
						  {

						          uint32_t code_item_len = dex_file->GetCodeItemSize(*code_item);
								  memset(dexfilepath,0,1000);
								  int size_int=(int)dex_file->Size();
								  uint32_t method_idx=artmethod->GetDexMethodIndex();
								  sprintf(dexfilepath,"/data/data/%s/corax/%d_ins_%d.bin",szProcName,size_int,(int)gettidv1());
								  int fp2=open(dexfilepath,O_CREAT|O_APPEND|O_RDWR,0666);
								  if(fp2>0){
									  lseek(fp2,0,SEEK_END);
									  memset(dexfilepath,0,1000);
									  int offset=artmethod->GetCodeItemOffset();
									  sprintf(dexfilepath,"{name:'%s',method_idx:%d,offset:%d,code_item_len:%d,ins:'",artmethod->PrettyMethod().c_str(),method_idx,offset,code_item_len);
									  int contentlength=0;
									  while(dexfilepath[contentlength]!=0) contentlength++;
									  result=write(fp2,(void*)dexfilepath,contentlength);
									  if(result<0)
												{
													LOG(ERROR) << "ArtMethod::dumpdexfilebyArtMethod,write ins file error";

													}
									  long outlen=0;
									  char* base64result=base64_encode((char*)code_item,(long)code_item_len,&outlen);
									  result=write(fp2,base64result,outlen);
									  if(result<0)
												{
													LOG(ERROR) << "ArtMethod::dumpdexfilebyArtMethod,write ins file error";

													}
									  result=write(fp2,"'};",3);
									  if(result<0)
												{
													LOG(ERROR) << "ArtMethod::dumpdexfilebyArtMethod,write ins file error";

													}
									  fsync(fp2);
									  close(fp2);
									  if(base64result!=nullptr){
										  free(base64result);
										  base64result=nullptr;
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

更改的部分有

1. open第二参数变成两个值  fcmdline = open(szCmdline, O_RDONLY | O_CREAT ,0644);
2. 诸如此类的路径更改 释放到app私有目录 更改文件名 规避检测sprintf(dexfilepath,"/data/data/%s/corax",szProcName);
3. codeitem长度的计算直接用一个函数 uint32_t code_item_len = dex_file->GetCodeItemSize(*code_item);
4. 查看是否存在变成这样access(dexfilepath,F_OK) !=-1 

导包如下

```
//add


#include <fcntl.h>

#include "sys/stat.h"
#define gettidv1() syscall(__NR_gettid)
//add
```

移植也差不多了。编译刷机

有了

![image-20240808062646545](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712337.png)

## 4.Dex重构

上面有讲到我们有三个文件，其中真正参与dex重构的是dex和bin，dex这时还是没有codeitem的

![image-20240808063004857](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712338.png)

而bin以json的格式 输出一个bin，里面存放了哪个函数的哪个offset对应哪个codeitem

![image-20240808063033100](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712339.png)

我们dex重构的意义也就是**把他codeitem按照一定的规则填写回去**

其实这也是有开源项目的，我们改改就能用

重要逻辑也就是这样

```
package com.android.dx.unpacker;

import com.android.dex.Dex;
import com.android.dex.util.FileUtils;
import com.android.dx.command.dexer.DxContext;
import com.android.dx.merge.CollisionPolicy;
import com.android.dx.merge.DexMerger;

import java.io.File;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.StandardCopyOption;

import javafx.scene.shape.Path;

class DexFixer {
    public static void main(String[] args) throws IOException {
        if (args.length < 3) {
            printUsage();
            return;
        }

        String dexpath = args[0];
        String binpath = args[1];
        String outpath = args[2];

        File dexfile = new File(dexpath);
        File binfile = new File(binpath);
        if (!dexfile.exists() || !binfile.exists()) {
            System.out.println("Usage: dexfile or binfile not found");
            return;
        }
        if (!dexfile.getPath().endsWith(".dex")) {
            System.out.println("Usage: DexFixer dexpath end not .dex");
            return;
        }

        Dex[] dexes = new Dex[1];
        dexes[0] = new Dex(dexfile);
        MethodCodeItemFile methodCodeItemFile = new MethodCodeItemFile(binfile);
        Dex merged = new DexMerger(dexes, CollisionPolicy.KEEP_FIRST, new DxContext(),
                methodCodeItemFile.getMethodCodeItems()).merge();
        merged.writeTo(new File(outpath));
        System.out.println("success")
    }

    private static void printUsage() {
        System.out.println("Usage: DexFixer unpacker output");
    }
}
```

简单来讲，有三个参数1.dex 2.bin 3.dex 第一个是空dex，第二个就是bin，第三个是最后重构的结果。

代码逻辑也就是利用bin传给一个自定义的MethodCodeItemFile对象，如下，对bin作解析给了类的成员

```
package com.android.dx.unpacker;

import com.android.dex.util.ByteInput;
import com.android.dex.util.ByteOutput;
import com.google.gson.Gson;

import java.io.ByteArrayOutputStream;
import java.io.File;
import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.IOException;
import java.io.InputStream;
import java.nio.ByteBuffer;
import java.nio.ByteOrder;
import java.util.Base64;
import java.util.HashMap;
import java.util.Map;

public class MethodCodeItemFile {
    private ByteBuffer data;
    private Map<Integer, MethodCodeItem> map;
    private String jsondata;

    public MethodCodeItemFile(File file) {
        try (InputStream inputStream = new FileInputStream(file)) {
            loadFrom(inputStream);
            this.map = new HashMap<>();
            Gson gson = new Gson();
            String[] items = this.jsondata.split(";");
            for (int i = 0; i < items.length; i++) {
                String codeJson = items[i];
                JsonCodeItem codedata = gson.fromJson(codeJson, JsonCodeItem.class);
                MethodCodeItem codeItem = new MethodCodeItem();
                codeItem.index = codedata.method_idx;
                codeItem.descriptor = codedata.name;
                codeItem.size = codedata.code_item_len;
                byte[] buff = Base64.getDecoder().decode(codedata.ins);
                codeItem.code = buff;
                this.map.put(codeItem.index, codeItem);
            }

//            while (data.hasRemaining())
//            {
//                MethodCodeItem codeItem = new MethodCodeItem();
//                codeItem.index = readInt();
//                codeItem.descriptor = readCString();
//                codeItem.size = readInt();
//                codeItem.code = readByteArray(codeItem.size);
//                this.map.put(codeItem.index, codeItem);
//            }
        } catch (Exception e) {
            System.out.println("Warn: " + file.getPath() + " maybe invalid format!");
            e.printStackTrace();
        }
    }

    public Map<Integer, MethodCodeItem> getMethodCodeItems() {
        return this.map;
    }

    public byte[] readByteArray(int length) {
        byte[] result = new byte[length];
        data.get(result);
        return result;
    }

    public int readInt() {
        return data.getInt();
    }

    public String readCString() {
        byte b;
        StringBuilder s = new StringBuilder("");
        do {
            b = data.get();
            if (b != 0) {
                s.append((char) b);
            }
        }
        while (b != 0);
        return String.valueOf(s);
    }

    private void loadFrom(InputStream in) throws IOException {
        ByteArrayOutputStream bytesOut = new ByteArrayOutputStream();
        byte[] buffer = new byte[8192];

        int count;
        while ((count = in.read(buffer)) != -1) {
            bytesOut.write(buffer, 0, count);
        }

//        this.data = ByteBuffer.wrap(bytesOut.toByteArray());
//        this.data.order(ByteOrder.LITTLE_ENDIAN);
        this.jsondata = bytesOut.toString();
    }
}

```

然后回到上面DexMerger 作重构

```
    private DexMerger(Dex[] dexes, CollisionPolicy collisionPolicy, DxContext context,
            WriterSizes writerSizes) throws IOException {
        this.dexes = dexes;
        this.collisionPolicy = collisionPolicy;
        this.context = context;
        this.writerSizes = writerSizes;

        dexOut = new Dex(writerSizes.size());

        indexMaps = new IndexMap[dexes.length];
        for (int i = 0; i < dexes.length; i++) {
            indexMaps[i] = new IndexMap(dexOut, dexes[i].getTableOfContents());
        }
        instructionTransformer = new InstructionTransformer();

        headerOut = dexOut.appendSection(writerSizes.header, "header");
        idsDefsOut = dexOut.appendSection(writerSizes.idsDefs, "ids defs");

        contentsOut = dexOut.getTableOfContents();
        contentsOut.dataOff = dexOut.getNextSectionStart();

        contentsOut.mapList.off = dexOut.getNextSectionStart();
        contentsOut.mapList.size = 1;
        mapListOut = dexOut.appendSection(writerSizes.mapList, "map list");

        contentsOut.typeLists.off = dexOut.getNextSectionStart();
        typeListOut = dexOut.appendSection(writerSizes.typeList, "type list");

        contentsOut.annotationSetRefLists.off = dexOut.getNextSectionStart();
        annotationSetRefListOut = dexOut.appendSection(
                writerSizes.annotationsSetRefList, "annotation set ref list");

        contentsOut.annotationSets.off = dexOut.getNextSectionStart();
        annotationSetOut = dexOut.appendSection(writerSizes.annotationsSet, "annotation sets");

        contentsOut.classDatas.off = dexOut.getNextSectionStart();
        classDataOut = dexOut.appendSection(writerSizes.classData, "class data");

        contentsOut.codes.off = dexOut.getNextSectionStart();
        codeOut = dexOut.appendSection(writerSizes.code, "code");

        contentsOut.stringDatas.off = dexOut.getNextSectionStart();
        stringDataOut = dexOut.appendSection(writerSizes.stringData, "string data");

        contentsOut.debugInfos.off = dexOut.getNextSectionStart();
        debugInfoOut = dexOut.appendSection(writerSizes.debugInfo, "debug info");

        contentsOut.annotations.off = dexOut.getNextSectionStart();
        annotationOut = dexOut.appendSection(writerSizes.annotation, "annotation");

        contentsOut.encodedArrays.off = dexOut.getNextSectionStart();
        encodedArrayOut = dexOut.appendSection(writerSizes.encodedArray, "encoded array");

        contentsOut.annotationsDirectories.off = dexOut.getNextSectionStart();
        annotationsDirectoryOut = dexOut.appendSection(
                writerSizes.annotationsDirectory, "annotations directory");

        contentsOut.dataSize = dexOut.getNextSectionStart() - contentsOut.dataOff;
    }

```

开源项目地址：https://github.com/nirvanawoody/DexFixer

输出的成果如下

该有的都有了

![image-20240808063829482](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712340.png)

## 5.Frida增强功能

为啥要用Frida增强功能呢？

因为有些壳在类加载器修正的时候，**会隐去明确的父子关系，又或者会在类设置某些垃圾函数**，这些垃圾函数被调用就退出进程，为了防范主动调用。为了脱下这些类，可以用**Frida指定对某些类进行脱壳**。

在小肩膀师傅的课程里面它把这里的线程给注释掉了，可能是为了看得更加清晰一点。但其实我觉得没必要，因为我们后续定义魔改的系统编译后**已经有我们的api**了，并且都是静态方法，我们甚至直接frida Java.choose就可以了

![image-20240808063855316](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712341.png)

但是！！**这里有个坑**搞了我一手，如果把上面注释掉的话，我们自己的savedex.java就不会参与编译，应该没添加上。我后续采用的方法就是把savedex.java的内容全部处理到ActivityThread里面去，然后编译就可以了。

先写个简单脚本试试

![image-20240808064317048](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712342.png)

是可以输出classloader的

![image-20240808064357362](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712343.png)

那我们直接调用我们之前的定义的coraxwithClassloader

现在这个目录没东西

![image-20240808064520928](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712344.png)

调用之后，就有了

![image-20240808064557540](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712345.png)

看一下bin，符合预期，我们传入的是dexclassloader里面有我们这个loader会加载的类

![image-20240808064630865](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712346.png)



再试试能不能单独脱某个类，脚本如下，也算好理解，反射获得dalvik.system.DexFile下面叫做saveMethodCode的函数（我们自己魔改的）然后设置好权限，通过loadClassAndInvoke脱com.xiaojianbang.encrypt.DES

![image-20240808064742234](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712347.png)

可以看到也可以成功

![image-20240808064943709](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712348.png)

![image-20240808065013887](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202408080712349.png)

大功告成

# 总结

本篇我们做了Fart移植到安卓10上面的研究学习，并成功进行了移植，刷机。然后看了一下dex重构的内容，以及如何用Frida增强Fart功能。