---
layout:     post
title:      【Android学习】Fla源码分析
subtitle:   工具学习
date:       2024-05-19
author:     Corax
header-img: img/post-bg-coffee1.jpg
catalog: true
tags:
    - 【Ollvm】
    - 【逆向】
    - 【Android】
---

# Fla源码分析

看完了Bcf火速赶往Fla啊

本篇博客是我根据vae师傅写的，找了别的博客，都没vae师傅这种跟随源代码查看实时混淆结果情况的做法，很认可这种思想，遂学习。

为了防止越学越迷茫，先看看Fla是怎么个过程

在我理解里面，本身的程序流程会使用各种各样的跳转指令更改控制流的效果，虽然有不少循环和回call，但总体而言流程还算是线性的，这样的代码强度其实对逆向者比较有利，无论是动态还是静态，我们都可以相对简单的了解整个程序从开始到结束。（只是相对）

而Fla，控制流平坦化的结果会变成啥样呢 （忽略歌词哈）

![](https://typora-1321221957.cos.ap-shanghai.myqcloud.com/image1/202405232225670.png)

我们的基本块都被扁平化了，比如执行完基本块1，有一个值进入switchcase语句（把这个部分就理解成分发器），然后到达下一个基本块。

而混淆的流程

```
1.收集原函数中所有的基本块，并初始化随机数种子。

2.对入口基本块进行处理，切分基本块保证入口基本块只有一个后继。

3.给每一个基本块分配一个随机数字，并新建一个变量var，在入口基本块中赋值为入口基本块后继基本块对应的数字。

4.构造出基本的switch结构和循环框架，使得switch链接所有原有基本块。

5.修正每个原有基本块的后继，使其跳转至switch结构，并在跳转之前根据后继和跳转条件构造对var的赋值语句。
```

那下面我们具体来看

# runOnFunction

```
bool Flattening::runOnFunction(Function &F) {
  Function *tmp = &F;
  // Do we obfuscate
  if (toObfuscate(flag, tmp, "fla")) {
    if (flatten(tmp)) {
      ++Flattened;
    }
  }

  return false;
}
```

pass当中经典的runonfunction，可以看到他也是先判断是不是fla，然后在执行flatten函数

# flatten

和bcf的结构不一样，这玩意基本都写在这个函数里面，没事，我们一点一点研究。

```
  for (Function::iterator i = f->begin(); i != f->end(); ++i) {
    BasicBlock *tmp = &*i;
    origBB.push_back(tmp);

    BasicBlock *bb = &*i;
    if (isa<InvokeInst>(bb->getTerminator())) {
      return false;
    }
  }//这里还是在划分基本快
```

划分基本块，推到origBB里面。

```
  if (origBB.size() <= 1) {
    return false;
  }
```

如果origBB，也就是里面一个基本块都么的，那就不用混淆了。

```
  // Remove first BB
  origBB.erase(origBB.begin());

  // Get a pointer on the first BB
  Function::iterator tmp = f->begin();  //++tmp;
  BasicBlock *insert = &*tmp;

  // If main begin with an if
  BranchInst *br = NULL;
  if (isa<BranchInst>(insert->getTerminator())) {
    br = cast<BranchInst>(insert->getTerminator());
  }

  if ((br != NULL && br->isConditional()) ||
      insert->getTerminator()->getNumSuccessors() > 1) {
    BasicBlock::iterator i = insert->end();
	--i;

    if (insert->size() > 1) {
      --i;
    }

    BasicBlock *tmpBB = insert->splitBasicBlock(i, "first");
    origBB.insert(origBB.begin(), tmpBB);
  }
```

我们把里面的第一个基本块，就是入口块拿出来单独处理。

关于这段我的解释呢：

```
首先会去看最后一条是不是分支条件指令
	如果是 看看后面会不会有多个后记基本块 或者是否为条件分支
		如果是则获取这个基本块最后一条指令的迭代器
	如果 基本块大小大于1，迭代器再前移一位
	再根据当前迭代器位置split成两个 第一个叫做first，新的基本块插入到 origBB 基本块的开头位置。
```

例如此时的初始代码

```
 %retval = alloca i32, align 4
  %argc.addr = alloca i32, align 4
  %argv.addr = alloca i8**, align 8
  %a = alloca i32, align 4
  store i32 0, i32* %retval, align 4
  store i32 %argc, i32* %argc.addr, align 4
  store i8** %argv, i8*** %argv.addr, align 8
  %0 = load i8**, i8*** %argv.addr, align 8
  %arrayidx = getelementptr inbounds i8*, i8** %0, i64 1
  %1 = load i8*, i8** %arrayidx, align 8
  %call = call i32 @atoi(i8* %1) #3
  store i32 %call, i32* %a, align 4
  %2 = load i32, i32* %a, align 4
  br label %NodeBlock8
```

然后经过这里

```
witchVar =
      new AllocaInst(Type::getInt32Ty(f->getContext()), 0, "switchVar", insert);
  new StoreInst(
      ConstantInt::get(Type::getInt32Ty(f->getContext()),
                       llvm::cryptoutils->scramble32(0, scrambling_key)),
      switchVar, insert);
```

是创建了一个switchvar变量，然后去[获取一个](https://so.csdn.net/so/search?q=获取一个&spm=1001.2101.3001.7020)随机整数创建store指令塞给switchvar中

## switchvar

也就是在switchvar添了如下这一行：

```
  store i32 157301900, i32* %switchVar
```

经过上面switchvar原始代码如下

```
entry:
  %.reg2mem = alloca i32
  %retval = alloca i32, align 4
  %argc.addr = alloca i32, align 4
  %argv.addr = alloca i8**, align 8
  %a = alloca i32, align 4
  store i32 0, i32* %retval, align 4
  store i32 %argc, i32* %argc.addr, align 4
  store i8** %argv, i8*** %argv.addr, align 8
  %0 = load i8**, i8*** %argv.addr, align 8
  %arrayidx = getelementptr inbounds i8*, i8** %0, i64 1
  %1 = load i8*, i8** %arrayidx, align 8
  %call = call i32 @atoi(i8* %1) #3
  store i32 %call, i32* %a, align 4
  %2 = load i32, i32* %a, align 4
  store i32 %2, i32* %.reg2mem
  %switchVar = alloca i32
```

## 创建正式的switch结构

```
  // Create main loop
  loopEntry = BasicBlock::Create(f->getContext(), "loopEntry", f, insert);
  loopEnd = BasicBlock::Create(f->getContext(), "loopEnd", f, insert);
```

结构如下 还没啥东西

```
loopEntry:                                      
 
loopEnd:                                          
```

在loopEntry里面新建一个load指令，并且把switchVar推进去

```
 load = new LoadInst(switchVar, "switchVar", loopEntry);
```

则

```
loopEntry:                                        ; preds = %entry, %loopEnd
  %switchVar10 = load i32, i32* %switchVar
```

## 移动firstbb

```
  // Move first BB on top
  insert->moveBefore(loopEntry);
  BranchInst::Create(loopEntry, insert);

  // loopEnd jump to loopEntry
  BranchInst::Create(loopEntry, loopEnd);
```

把insert插入到loopEntry之前，这里的insert就是entry基本块，再创建两个跳转指令，从insert跳转到loopentry 从loopend跳到loopentry

此时的entry模块

```
entry:
  %.reg2mem = alloca i32
  %retval = alloca i32, align 4
  %argc.addr = alloca i32, align 4
  %argv.addr = alloca i8**, align 8
  %a = alloca i32, align 4
  store i32 0, i32* %retval, align 4
  store i32 %argc, i32* %argc.addr, align 4
  store i8** %argv, i8*** %argv.addr, align 8
  %0 = load i8**, i8*** %argv.addr, align 8
  %arrayidx = getelementptr inbounds i8*, i8** %0, i64 1
  %1 = load i8*, i8** %arrayidx, align 8
  %call = call i32 @atoi(i8* %1) #3
  store i32 %call, i32* %a, align 4
  %2 = load i32, i32* %a, align 4
  store i32 %2, i32* %.reg2mem
  %switchVar = alloca i32
  store i32 157301900, i32* %switchVar
  br label %loopEntry
```

可以看到最后来到了loopEntry

而loopend也基本完成任务

```
loopEnd:                                        
br label %loopEntry
```

## 创建基本块

```
  BasicBlock *swDefault =
      BasicBlock::Create(f->getContext(), "switchDefault", f, loopEnd);
  BranchInst::Create(loopEnd, swDefault);
```

这个叫做switchdefault 记住他

其中的指令

```
switchDefault:                                    ; preds = %loopEntry
  br label %loopEnd
```

源码指令 创建一个switch指令，位置是在loopentry基本块下，且创建了0个case，然后设置了条件为load，就上面的load。

```
  // Create switch instruction itself and set condition
  switchI = SwitchInst::Create(&*f->begin(), swDefault, 0, loopEntry);
  switchI->setCondition(load);
```

把entry最后一行跳转指令删除后再创建了一个跳转指令，**从entry跳转到loopentry**

```
  // Remove branch jump from 1st BB and make a jump to the while
  f->begin()->getTerminator()->eraseFromParent();

  BranchInst::Create(loopEntry, &*f->begin());

  // Put all BB in the switch
  for (vector<BasicBlock *>::iterator b = origBB.begin(); b != origBB.end();
       ++b) {
    BasicBlock *i = *b;
    ConstantInt *numCase = NULL;

    // Move the BB inside the switch (only visual, no code logic)
    i->moveBefore(loopEnd);

    // Add case to switch
    numCase = cast<ConstantInt>(ConstantInt::get(
        switchI->getCondition()->getType(),
        llvm::cryptoutils->scramble32(switchI->getNumCases(), scrambling_key)));
    switchI->addCase(numCase, i);
  }
```

那现在初始代码变成了这样

```
entry:
  %.reg2mem = alloca i32
  %retval = alloca i32, align 4
  %argc.addr = alloca i32, align 4
  %argv.addr = alloca i8**, align 8
  %a = alloca i32, align 4
  store i32 0, i32* %retval, align 4
  store i32 %argc, i32* %argc.addr, align 4
  store i8** %argv, i8*** %argv.addr, align 8
  %0 = load i8**, i8*** %argv.addr, align 8
  %arrayidx = getelementptr inbounds i8*, i8** %0, i64 1
  %1 = load i8*, i8** %arrayidx, align 8
  %call = call i32 @atoi(i8* %1) #3
  store i32 %call, i32* %a, align 4
  %2 = load i32, i32* %a, align 4
  store i32 %2, i32* %.reg2mem
  %switchVar = alloca i32
  store i32 157301900, i32* %switchVar
  br label %loopEntry

loopEntry:     
%switchVar10 = load i32, i32* %switchVar
  switch i32 %switchVar10, label %switchDefault [
  ]

switchDefault:                                    ; preds = %loopEntry
  br label %loopEnd

loopEnd:                                        
br label %loopEntry

```

## 创建case

接下来就需要各种各样的case咯

```
  for (std::vector<BasicBlock *>::iterator b = origBB.begin();
       b != origBB.end(); ++b) {
    BasicBlock *i = *b;
    ConstantInt *numCase = NULL;

    // Move the BB inside the switch (only visual, no code logic)
    i->moveBefore(loopEnd);

    // Add case to switch
    numCase = cast<ConstantInt>(ConstantInt::get(
        switchI->getCondition()->getType(),
        llvm::cryptoutils->scramble32(switchI->getNumCases(), scrambling_key)));
    switchI->addCase(numCase, i);
  }
```

把block推进去之后添加case值 就是case分支里面的case值，这个值它是随机生成的，种子的话是Entry.cpp里面的那个AesSeed值，如果确定AesSeed的话，那么这里随机生成的case每次都是固定的。

switchI->addCase(numCase, i);紧接着在switch里面增加一个case值，跳转到NodeBlock8里面。
目前switch执行完一次后，loopentry基本bolck块如下：

```
loopEntry:     
%switchVar10 = load i32, i32* %switchVar
  switch i32 %switchVar10, label %switchDefault [
   i32 157301900, label %NodeBlock8
  ]

```

这里结束之后的初始代码

```
entry:
  %retval = alloca i32, align 4
  %argc.addr = alloca i32, align 4
  %argv.addr = alloca i8**, align 8
  %a = alloca i32, align 4
  store i32 0, i32* %retval, align 4
  store i32 %argc, i32* %argc.addr, align 4
  store i8** %argv, i8*** %argv.addr, align 8
  %0 = load i8**, i8*** %argv.addr, align 8
  %arrayidx = getelementptr inbounds i8*, i8** %0, i64 1
  %1 = load i8*, i8** %arrayidx, align 8
  %call = call i32 @atoi(i8* %1) #3
  store i32 %call, i32* %a, align 4
  %2 = load i32, i32* %a, align 4
  br label %NodeBlock8

NodeBlock8:                                       ; preds = %entry
  %Pivot9 = icmp slt i32 %2, 2
  br i1 %Pivot9, label %LeafBlock, label %NodeBlock

NodeBlock:                                        ; preds = %NodeBlock8
  %Pivot = icmp slt i32 %2, 3
  br i1 %Pivot, label %sw.bb2, label %LeafBlock6

LeafBlock6:                                       ; preds = %NodeBlock
  %SwitchLeaf7 = icmp eq i32 %2, 3
  br i1 %SwitchLeaf7, label %sw.bb4, label %NewDefault

LeafBlock:                                        ; preds = %NodeBlock8
  %SwitchLeaf = icmp eq i32 %2, 1
  br i1 %SwitchLeaf, label %sw.bb, label %NewDefault

sw.bb:                                            ; preds = %LeafBlock
  %call1 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([5 x i8], [5 x i8]* @.str, i64 0, i64 0))
  br label %sw.epilog

sw.bb2:                                           ; preds = %NodeBlock
  %call3 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([5 x i8], [5 x i8]* @.str.1, i64 0, i64 0))
  br label %sw.epilog

sw.bb4:                                           ; preds = %LeafBlock6
  %call5 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([5 x i8], [5 x i8]* @.str.2, i64 0, i64 0))
  br label %sw.epilog

NewDefault:                                       ; preds = %LeafBlock6, %LeafBlock
  br label %sw.default

sw.default:                                       ; preds = %NewDefault
  br label %sw.epilog

sw.epilog:                                        ; preds = %sw.default, %sw.bb4, %sw.bb2, %sw.bb
  %3 = load i32, i32* %a, align 4
  %cmp = icmp eq i32 %3, 0
  br i1 %cmp, label %if.then, label %if.else

if.then:                                          ; preds = %sw.epilog
  store i32 1, i32* %retval, align 4
  br label %return

if.else:                                          ; preds = %sw.epilog
  store i32 10, i32* %retval, align 4
  br label %return

return:                                           ; preds = %if.else, %if.then
  %4 = load i32, i32* %retval, align 4
  ret i32 %4
  
loopEnd:                                          ; preds = %if.else, %if.then, %sw.epilog, %sw.default, %NewDefault, %sw.bb4, %sw.bb2, %sw.bb, %LeafBlock, %LeafBlock6, %NodeBlock, %NodeBlock8, %switchDefault
  br label %loopEntry

}
```

可以看到有些雏形了

## 枚举更改各个case block块

```
   if (i->getTerminator()->getNumSuccessors() == 0) {
      continue;
    }
```

getNumSuccessors是获取后续BB的个数，Ret BB后继BB为0个（判断分支），直接continue

## 不是条件跳转

```
    // If it's a non-conditional jump
    if (i->getTerminator()->getNumSuccessors() == 1) {
      // Get successor and delete terminator
      BasicBlock *succ = i->getTerminator()->getSuccessor(0);
      i->getTerminator()->eraseFromParent();

      // Get next case
      numCase = switchI->findCaseDest(succ);

      // If next case == default case (switchDefault)
      if (numCase == NULL) {
        numCase = cast<ConstantInt>(
            ConstantInt::get(switchI->getCondition()->getType(),
                             llvm::cryptoutils->scramble32(
                                 switchI->getNumCases() - 1, scrambling_key)));
      }

      // Update switchVar and jump to the end of loop
      new StoreInst(numCase, load->getPointerOperand(), i);
      BranchInst::Create(loopEnd, i);
      continue;
    }
```

先判断分支是否能够找到，不为null后先去根据原来条件去创建一个store指令，然后创建一个跳转指令跳转到loopend，再把原来跳转指令抹去。

原

```
sw.bb:                                            ; preds = %LeafBlock
  %call1 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([5 x i8], [5 x i8]* @.str, i64 0, i64 0))
  br label %sw.epilog
```

后

```
sw.bb:                                            ; preds = %LeafBlock
  %call1 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([5 x i8], [5 x i8]* @.str, i64 0, i64 0))
  store i32 387774014, i32* %switchVar
  br label %loopEnd
```

## 对于条件跳转 

```
  if (i->getTerminator()->getNumSuccessors() == 2) {
      // Get next cases
      ConstantInt *numCaseTrue =
          switchI->findCaseDest(i->getTerminator()->getSuccessor(0));
      ConstantInt *numCaseFalse =
          switchI->findCaseDest(i->getTerminator()->getSuccessor(1));

      // Check if next case == default case (switchDefault)
      if (numCaseTrue == NULL) {
        numCaseTrue = cast<ConstantInt>(
            ConstantInt::get(switchI->getCondition()->getType(),
                             llvm::cryptoutils->scramble32(
                                 switchI->getNumCases() - 1, scrambling_key)));
      }

      if (numCaseFalse == NULL) {
        numCaseFalse = cast<ConstantInt>(
            ConstantInt::get(switchI->getCondition()->getType(),
                             llvm::cryptoutils->scramble32(
                                 switchI->getNumCases() - 1, scrambling_key)));
      }

      // Create a SelectInst
      BranchInst *br = cast<BranchInst>(i->getTerminator());
      SelectInst *sel =
          SelectInst::Create(br->getCondition(), numCaseTrue, numCaseFalse, "",
                             i->getTerminator());

      // Erase terminator
      i->getTerminator()->eraseFromParent();

      // Update switchVar and jump to the end of loop
      new StoreInst(sel, load->getPointerOperand(), i);
      BranchInst::Create(loopEnd, i);
      continue;
    }

```

首先会把两个跳转分支都取出来，先判断两个分支是否都能够找到，如果都不为null 的话，那么**取出原来的跳转指令，根据br的两个分支条件，去创建一个SelectInst**然后**再删除原来指令**，创建一个store指令，再去创建一个跳转指令跳转到loopend。

比如 

原

```
NodeBlock8:                                       ; preds = %entry
  %Pivot9 = icmp slt i32 %2, 2
  br i1 %Pivot9, label %LeafBlock, label %NodeBlock
 
```

后

```
NodeBlock8:                                       ; preds = %entry
  %Pivot9 = icmp slt i32 %2, 2
  %3 = select i1 %Pivot9, i32 -1519555718, i32 241816174
  store i32 %3, i32* %switchVar
  br label %loopEnd
```

```
define dso_local i32 @main(i32 %argc, i8** %argv) #0 {
entry:
  %.reg2mem = alloca i32
  %retval = alloca i32, align 4
  %argc.addr = alloca i32, align 4
  %argv.addr = alloca i8**, align 8
  %a = alloca i32, align 4
  store i32 0, i32* %retval, align 4
  store i32 %argc, i32* %argc.addr, align 4
  store i8** %argv, i8*** %argv.addr, align 8
  %0 = load i8**, i8*** %argv.addr, align 8
  %arrayidx = getelementptr inbounds i8*, i8** %0, i64 1
  %1 = load i8*, i8** %arrayidx, align 8
  %call = call i32 @atoi(i8* %1) #3
  store i32 %call, i32* %a, align 4
  %2 = load i32, i32* %a, align 4
  store i32 %2, i32* %.reg2mem
  %switchVar = alloca i32
  store i32 157301900, i32* %switchVar
  br label %loopEntry

loopEntry:                                        ; preds = %entry, %loopEnd
  %switchVar10 = load i32, i32* %switchVar
  switch i32 %switchVar10, label %switchDefault [
    i32 157301900, label %NodeBlock8
    i32 241816174, label %NodeBlock
    i32 1003739776, label %LeafBlock6
    i32 -1519555718, label %LeafBlock
    i32 -749093422, label %sw.bb
    i32 1599617141, label %sw.bb2
    i32 1815329037, label %sw.bb4
    i32 1738940479, label %NewDefault
    i32 -282945350, label %sw.default
    i32 387774014, label %sw.epilog
    i32 1681741611, label %if.then
    i32 347219667, label %if.else
    i32 -618048859, label %return
  ]

switchDefault:                                    ; preds = %loopEntry
  br label %loopEnd

NodeBlock8:                                       ; preds = %entry
  %Pivot9 = icmp slt i32 %2, 2
  %3 = select i1 %Pivot9, i32 -1519555718, i32 241816174
  store i32 %3, i32* %switchVar
  br label %loopEnd

NodeBlock:                                        ; preds = %NodeBlock8
  %Pivot = icmp slt i32 %2, 3
  %4 = select i1 %Pivot, i32 1599617141, i32 1003739776
  store i32 %4, i32* %switchVar
  br label %loopEnd

LeafBlock6:                                       ; preds = %NodeBlock
  %SwitchLeaf7 = icmp eq i32 %2, 3
  %5 = select i1 %SwitchLeaf7, i32 1815329037, i32 1738940479
  store i32 %5, i32* %switchVar
  br label %loopEnd

LeafBlock:                                        ; preds = %NodeBlock8
  %SwitchLeaf = icmp eq i32 %2, 1
  %6 = select i1 %SwitchLeaf, i32 -749093422, i32 1738940479
  store i32 %6, i32* %switchVar
  br label %loopEnd

sw.bb:                                            ; preds = %LeafBlock
  %call1 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([5 x i8], [5 x i8]* @.str, i64 0, i64 0))
  store i32 387774014, i32* %switchVar
  br label %loopEnd

sw.bb2:                                           ; preds = %NodeBlock
  %call3 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([5 x i8], [5 x i8]* @.str.1, i64 0, i64 0))
  br label %sw.epilog

sw.bb4:                                           ; preds = %LeafBlock6
  %call5 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([5 x i8], [5 x i8]* @.str.2, i64 0, i64 0))
  store i32 387774014, i32* %switchVar
  br label %loopEnd

NewDefault:                                       ; preds = %LeafBlock6, %LeafBlock
  br label %sw.default

sw.default:                                       ; preds = %NewDefault
  store i32 387774014, i32* %switchVar
  br label %loopEnd

sw.epilog:                                        ; preds = %sw.default, %sw.bb4, %sw.bb2, %sw.bb
  %3 = load i32, i32* %a, align 4
  %cmp = icmp eq i32 %3, 0
  %8 = select i1 %cmp, i32 1681741611, i32 347219667
  store i32 %8, i32* %switchVar
  br label %loopEnd

if.then:                                          ; preds = %sw.epilog
  store i32 1, i32* %retval, align 4
  store i32 -618048859, i32* %switchVar
  br label %loopEnd

if.else:                                          ; preds = %sw.epilog
  store i32 10, i32* %retval, align 4
  store i32 -618048859, i32* %switchVar
  br label %loopEnd

return:                                           ; preds = %if.else, %if.then
  %4 = load i32, i32* %retval, align 4
  ret i32 %4

loopEnd:                                          ; preds = %if.else, %if.then, %sw.epilog, %sw.default, %NewDefault, %sw.bb4, %sw.bb2, %sw.bb, %LeafBlock, %LeafBlock6, %NodeBlock, %NodeBlock8, %switchDefault
  br label %loopEntry
}

```

初始代码变成这样啦，也差不多咯































