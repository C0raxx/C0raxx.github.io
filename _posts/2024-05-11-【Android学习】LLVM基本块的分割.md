---
layout:     post
title:      【Android学习】基本块的分割
subtitle:   工具学习
date:       2024-05-11
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【Ollvm】
    - 【逆向】
    - 【Android】
---

# LLVM 基本块的分割



其实一直不是很理解LLVM是如何对函数体中的函数块进行划分的，不知道用的是什么规则

## 理论

基本块分割就是将基本块分割成等价的若干基本块，分割之后在基本块中间加上一些跳转语句。

![image-20240521172405388](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202405211800559.png)

我们这边是自己对这些基本块进行分割，过程是

```
遍历函数中的每个基本块
判断每个基本块当中有没有phi 有就跳过
没的话 根据配置进行划分
```

### 关于PHI

由SSA形式衍生而来，具体phi 指令根据“在当前 block 之前执行的是哪一个 predecessor(前任) block”来得到相应的值

简单来说phi指令的作用就是使程序走到希望的程序流程当中。

先这样子 后面深入了解

## 要用的东西

- 额外参数指定：读取opt指令中的自定义参数，来确定我们要把一个块分割成几个块
- splitBasicBlock()：将一个基本块从特定的地方分割为两个基本块
- isa<>函数：判断一个数据是不是我们想要的类型

### 额外参数指定

就是字面意思，我们为了在执行指令的时候可以灵活控制分割基本块代码中的参数（就例如分割成多少基本块）

```
#include "llvm/Support/CommandLine.h"
//可选的参数，指定一个基本块会被分裂成几个基本块，默认值为2
static cl::opt<int> splitNum( "split_num", cl::init(2), cl::desc( "Split <split_num> time(s) each BB"));
```

其中的split_num默认为init 2，也就是默认为2，如果我们没有自定义，那么就是2了。

这么用的

```
opt -load ../Build/LLVMObfuscator.so -split -split_num 5 -S TestProgram.ll -o TestProgram_split.ll
```

### splitBasicBlock()

这玩意是basicblock类中的一个成员函数

```
// Split the basic block into two basic blocks at the specified instruction.
  ///
  /// If \p Before is true, splitBasicBlockBefore handles the
  /// block splitting. Otherwise, execution proceeds as described below.
  ///
  /// Note that all instructions BEFORE the specified iterator
  /// stay as part of the original basic block, an unconditional branch is added
  /// to the original BB, and the rest of the instructions in the BB are moved
  /// to the new BB, including the old terminator.  The newly formed basic block
  /// is returned. This function invalidates the specified iterator.
  ///
  /// Note that this only works on well formed basic blocks (must have a
  /// terminator), and \p 'I' must not be the end of instruction list (which
  /// would cause a degenerate basic block to be formed, having a terminator
  /// inside of the basic block).
  ///
  /// Also note that this doesn't preserve any passes. To split blocks while
  /// keeping loop information consistent, use the SplitBlock utility function.
  BasicBlock *splitBasicBlock(iterator I, const Twine &BBName = "",
                              bool Before = false);
  BasicBlock *splitBasicBlock(Instruction *I, const Twine &BBName = "",
                              bool Before = false) {
    return splitBasicBlock(I->getIterator(), BBName, Before);
  }
```

可以看到两种用法

一个是使用迭代器iterator，另外的是instruction指针类型

​	我们用的是2，在需要的指令处进行划分。详细来讲就是将原先的基本块在指令I处一分为二，I之前的指令会放在第一个基本块内，之后的会放在第二个基本块内，随后会建立一个从第一个基本块到第二个基本块的绝对跳转。

### isa<>函数

isa<> 是一个模板函数，用于判断一个指针指向的数据的类型是不是给定的类型

这里我们用来看基本块有无phi指令

```
bool SplitBasicBlock::containsPHI(BasicBlock *BB)
{
    for (Instruction &I : *BB)
    {
        if (isa<PHINode>(&I))
        {
            return true;
        }
    }
    return false;
}
```

## 代码

写个头文件

```
#ifndef _SPLIT_BASIC_BLOCK_H
#define _SPLIT_BASIC_BLOCK_H

#include "llvm/IR/Function.h"
#include "llvm/Pass.h"

namespace llvm
{
    FunctionPass* createSplitBasicBlockPass();
}
```

主代码

```
#include "llvm/Support/raw_ostream.h"
#include "llvm/Support/CommandLine.h"
#include "llvm/IR/Instructions.h"
#include <vector>
using namespace std;
using namespace llvm;

// 可选的参数，指定一个基本块会被分裂成几个基本块，默认值为 2
static cl::opt <int> splitNum("split_num", cl::init(2), cl::desc("Split <split_num> time(s) each BB"));

namespace
{

    class SplitBasicBlock : public FunctionPass
    {
        public:
            static char ID;
            SplitBasicBlock() : FunctionPass(ID) {}

            bool runOnFunction(Function &F);
            
            // 判断一个基本块中是否包含 PHI指令(PHINode)
            bool containsPHI(BasicBlock *BB);

            // 对单个基本块执行分裂操作
            void split(BasicBlock *BB);

    };
    
}


bool SplitBasicBlock::runOnFunction(Function &F)
{

    // 用vector保存最初的基本块，为了不影响后续的遍历等操作
	vector <BasicBlock*> origBB;
    for (BasicBlock &BB : F)
    {
        origBB.push_back(&BB);
    }
    // 对每个不包含 PHI 指令的基本块执行分裂操作
    for(BasicBlock *BB : origBB)
    {
        if (!containsPHI(BB))
        {
            split(BB);
        }
    }
    return true;
}

bool SplitBasicBlock::containsPHI(BasicBlock *BB)
{
    for (Instruction &I : *BB)
    {
        if (isa<PHINode>(&I)) return true;
    }
    return false;
}

void SplitBasicBlock::split(BasicBlock *BB)
{
    // 计算分裂后每个基本块的大小：原基本块的大小 / 分裂数目（向上取整）
    int splitSize = (BB -> size() + splitNum - 1) / splitNum;
    BasicBlock *curBB = BB;
    for (int i = 1; i <= splitNum - 1; ++i)
    {
        int cnt = 0;
        for (Instruction &I : *curBB)
        {
            if (cnt++ == splitSize)
            {
                // 在 I 指令处对基本块进行分割
                curBB = curBB -> splitBasicBlock(&I);
                break;
            }
        }
    }

}
char SplitBasicBlock::ID = 0; // 初始化ID

static RegisterPass<SplitBasicBlock> X("split", "Split One Basic Block into Multiple Blocks.");
```

运行之后的.ll

```
; ModuleID = 'IR/TestProgram.ll'
source_filename = "TestProgram.cpp"
target datalayout = "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-unknown-linux-gnu"

@input = dso_local global [100 x i8] zeroinitializer, align 16
@enc = dso_local global <{ [22 x i8], [78 x i8] }> <{ [22 x i8] c"\86\8A}\87\93\8BM\81\80\8AC\7FII\86q\7FbSi(\9D", [78 x i8] zeroinitializer }>, align 16
@.str = private unnamed_addr constant [25 x i8] c"Please input your flag: \00", align 1
@.str.1 = private unnamed_addr constant [3 x i8] c"%s\00", align 1
@.str.2 = private unnamed_addr constant [18 x i8] c"Congratulations~\0A\00", align 1
@.str.3 = private unnamed_addr constant [18 x i8] c"Sorry try again.\0A\00", align 1

; Function Attrs: noinline nounwind optnone uwtable mustprogress
define dso_local void @_Z7encryptPhPc(i8* %0, i8* %1) #0 {
  %3 = alloca i8*, align 8
  %4 = alloca i8*, align 8
  %5 = alloca i32, align 4
  %6 = alloca i32, align 4
  store i8* %0, i8** %3, align 8
  store i8* %1, i8** %4, align 8
  br label %7

7:                                                ; preds = %2
  %8 = load i8*, i8** %4, align 8
  %9 = call i64 @strlen(i8* %8) #5
  %10 = trunc i64 %9 to i32
  store i32 %10, i32* %5, align 4
  store i32 0, i32* %6, align 4
  br label %11

11:                                               ; preds = %37, %7
  %12 = load i32, i32* %6, align 4
  %13 = load i32, i32* %5, align 4
  br label %14

14:                                               ; preds = %11
  %15 = icmp slt i32 %12, %13
  br i1 %15, label %16, label %38

16:                                               ; preds = %14
  %17 = load i8*, i8** %4, align 8
  %18 = load i32, i32* %6, align 4
  %19 = sext i32 %18 to i64
  %20 = getelementptr inbounds i8, i8* %17, i64 %19
  %21 = load i8, i8* %20, align 1
  %22 = sext i8 %21 to i32
  %23 = load i32, i32* %6, align 4
  %24 = sub nsw i32 32, %23
  %25 = add nsw i32 %22, %24
  br label %26

26:                                               ; preds = %16
  %27 = load i32, i32* %6, align 4
  %28 = xor i32 %25, %27
  %29 = trunc i32 %28 to i8
  %30 = load i8*, i8** %3, align 8
  %31 = load i32, i32* %6, align 4
  %32 = sext i32 %31 to i64
  %33 = getelementptr inbounds i8, i8* %30, i64 %32
  store i8 %29, i8* %33, align 1
  br label %34

34:                                               ; preds = %26
  %35 = load i32, i32* %6, align 4
  %36 = add nsw i32 %35, 1
  br label %37

37:                                               ; preds = %34
  store i32 %36, i32* %6, align 4
  br label %11, !llvm.loop !2

38:                                               ; preds = %14
  ret void
}

; Function Attrs: nounwind readonly willreturn
declare dso_local i64 @strlen(i8*) #1

; Function Attrs: noinline norecurse optnone uwtable mustprogress
define dso_local i32 @main(i32 %0, i8** %1) #2 {
  %3 = alloca i32, align 4
  %4 = alloca i32, align 4
  %5 = alloca i8**, align 8
  %6 = alloca [100 x i8], align 16
  %7 = alloca i8, align 1
  store i32 0, i32* %3, align 4
  store i32 %0, i32* %4, align 4
  store i8** %1, i8*** %5, align 8
  %8 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([25 x i8], [25 x i8]* @.str, i64 0, i64 0))
  br label %9

9:                                                ; preds = %2
  %10 = call i32 (i8*, ...) @__isoc99_scanf(i8* getelementptr inbounds ([3 x i8], [3 x i8]* @.str.1, i64 0, i64 0), i8* getelementptr inbounds ([100 x i8], [100 x i8]* @input, i64 0, i64 0))
  %11 = bitcast [100 x i8]* %6 to i8*
  call void @llvm.memset.p0i8.i64(i8* align 16 %11, i8 0, i64 100, i1 false)
  %12 = getelementptr inbounds [100 x i8], [100 x i8]* %6, i64 0, i64 0
  call void @_Z7encryptPhPc(i8* %12, i8* getelementptr inbounds ([100 x i8], [100 x i8]* @input, i64 0, i64 0))
  %13 = call i64 @strlen(i8* getelementptr inbounds ([100 x i8], [100 x i8]* @input, i64 0, i64 0)) #5
  %14 = icmp eq i64 %13, 22
  br i1 %14, label %15, label %21

15:                                               ; preds = %9
  %16 = getelementptr inbounds [100 x i8], [100 x i8]* %6, i64 0, i64 0
  %17 = call i32 @memcmp(i8* %16, i8* getelementptr inbounds ([100 x i8], [100 x i8]* bitcast (<{ [22 x i8], [78 x i8] }>* @enc to [100 x i8]*), i64 0, i64 0), i64 22) #5
  %18 = icmp ne i32 %17, 0
  br label %19

19:                                               ; preds = %15
  %20 = xor i1 %18, true
  br label %21

21:                                               ; preds = %19, %9
  %22 = phi i1 [ false, %9 ], [ %20, %19 ]
  %23 = zext i1 %22 to i8
  store i8 %23, i8* %7, align 1
  %24 = load i8, i8* %7, align 1
  %25 = trunc i8 %24 to i1
  br i1 %25, label %26, label %29

26:                                               ; preds = %21
  %27 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([18 x i8], [18 x i8]* @.str.2, i64 0, i64 0))
  br label %28

28:                                               ; preds = %26
  br label %32

29:                                               ; preds = %21
  %30 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([18 x i8], [18 x i8]* @.str.3, i64 0, i64 0))
  br label %31

31:                                               ; preds = %29
  br label %32

32:                                               ; preds = %31, %28
  %33 = load i32, i32* %3, align 4
  br label %34

34:                                               ; preds = %32
  ret i32 %33
}

declare dso_local i32 @printf(i8*, ...) #3

declare dso_local i32 @__isoc99_scanf(i8*, ...) #3

; Function Attrs: argmemonly nofree nosync nounwind willreturn writeonly
declare void @llvm.memset.p0i8.i64(i8* nocapture writeonly, i8, i64, i1 immarg) #4

; Function Attrs: nounwind readonly willreturn
declare dso_local i32 @memcmp(i8*, i8*, i64) #1

attributes #0 = { noinline nounwind optnone uwtable mustprogress "disable-tail-calls"="false" "frame-pointer"="all" "less-precise-fpmad"="false" "min-legal-vector-width"="0" "no-infs-fp-math"="false" "no-jump-tables"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="true" "stack-protector-buffer-size"="8" "target-cpu"="x86-64" "target-features"="+cx8,+fxsr,+mmx,+sse,+sse2,+x87" "tune-cpu"="generic" "unsafe-fp-math"="false" "use-soft-float"="false" }
attributes #1 = { nounwind readonly willreturn "disable-tail-calls"="false" "frame-pointer"="all" "less-precise-fpmad"="false" "no-infs-fp-math"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="true" "stack-protector-buffer-size"="8" "target-cpu"="x86-64" "target-features"="+cx8,+fxsr,+mmx,+sse,+sse2,+x87" "tune-cpu"="generic" "unsafe-fp-math"="false" "use-soft-float"="false" }
attributes #2 = { noinline norecurse optnone uwtable mustprogress "disable-tail-calls"="false" "frame-pointer"="all" "less-precise-fpmad"="false" "min-legal-vector-width"="0" "no-infs-fp-math"="false" "no-jump-tables"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="true" "stack-protector-buffer-size"="8" "target-cpu"="x86-64" "target-features"="+cx8,+fxsr,+mmx,+sse,+sse2,+x87" "tune-cpu"="generic" "unsafe-fp-math"="false" "use-soft-float"="false" }
attributes #3 = { "disable-tail-calls"="false" "frame-pointer"="all" "less-precise-fpmad"="false" "no-infs-fp-math"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="true" "stack-protector-buffer-size"="8" "target-cpu"="x86-64" "target-features"="+cx8,+fxsr,+mmx,+sse,+sse2,+x87" "tune-cpu"="generic" "unsafe-fp-math"="false" "use-soft-float"="false" }
attributes #4 = { argmemonly nofree nosync nounwind willreturn writeonly }
attributes #5 = { nounwind readonly willreturn }

!llvm.module.flags = !{!0}
!llvm.ident = !{!1}

!0 = !{i32 1, !"wchar_size", i32 4}
!1 = !{!"clang version 12.0.1"}
!2 = distinct !{!2, !3}
!3 = !{!"llvm.loop.mustprogress"}

```

是多了很多基本块









































































