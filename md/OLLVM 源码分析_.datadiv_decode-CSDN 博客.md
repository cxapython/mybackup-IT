> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/zhoubobobo/article/details/82178464)

                                                            

文章转载自：[http://www.ench4nt3r.com/2018/02/26/post/](http://www.ench4nt3r.com/2018/02/26/post/)

           

#### [](#分割基本块 "分割基本块")分割基本块

```
-mllvm -split: activates basic block splitting. Improve the flattening when applied together.
-mllvm -split_num=3: if the pass is activated, applies it 3 times on each basic block. Default: 1

```

ollvm 的每个混淆 pass 都是继承`FunctionPass`，所以他们的入口函数都是`runOnFunction`。

```
bool SplitBasicBlock::runOnFunction(Function &F) {
  if (!((SplitNum > 1) && (SplitNum <= 10))) {
    return false;
  }

  Function *tmp = &F;

  if (toObfuscate(flag, tmp, "split")) {
    split(tmp);
    ++Split;
  }

  return false;
}

```

`toObfuscate`函数通过查找`Functions annotations`和`flag`来判断是否启用了`split`。

```
void SplitBasicBlock::split(Function *f) {
  std::vector<BasicBlock *> origBB;
  int splitN = SplitNum;

  // Save all basic blocks
  for (Function::iterator I = f->begin(), IE = f->end(); I != IE; ++I) {
    origBB.push_back(&*I);
  }

```

首先保存所有的基本块。

```
for (std::vector<BasicBlock *>::iterator I = origBB.begin(),
                                         IE = origBB.end();
     I != IE; ++I) {
  BasicBlock *curr = *I;

  // No need to split a 1 inst bb
  // Or ones containing a PHI node
  if (curr->size() < 2 || containsPHI(curr)) {
    continue;
  }

  // Check splitN and current BB size
  if ((size_t)splitN > curr->size()) {
    splitN = curr->size() - 1;
  }

```

接着遍历所有基本块。如果基本块只有一条指令或者包含 PHI 结点的，不分割该基本块。

```
// Generate splits point
std::vector<int> test;
for (unsigned i = 1; i < curr->size(); ++i) {
  test.push_back(i);
}

// Shuffle
if (test.size() != 1) {
  shuffle(test);
  std::sort(test.begin(), test.begin() + splitN);
}

```

生成基本块的分割点。

```
  // Split
  BasicBlock::iterator it = curr->begin();
  BasicBlock *toSplit = curr;
  int last = 0;
  for (int i = 0; i < splitN; ++i) {
    for (int j = 0; j < test[i] - last; ++j) {
      ++it;
    }
    last = test[i];
    if(toSplit->size() < 2)
      continue;
    toSplit = toSplit->splitBasicBlock(it, toSplit->getName() + ".split");
  }

  ++Split;
}

```

最后调用`splitBasicBlock`分割基本块。

#### [](#控制流平坦化 "控制流平坦化")控制流平坦化

```
-mllvm -fla: activates control flow flattening

```

入口函数仍然是`runOnFunction`：

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

```
bool Flattening::flatten(Function *f) {
  vector<BasicBlock *> origBB;
  BasicBlock *loopEntry;
  BasicBlock *loopEnd;
  LoadInst *load;
  SwitchInst *switchI;
  AllocaInst *switchVar;

  // SCRAMBLER
  char scrambling_key[16];
  llvm::cryptoutils->get_bytes(scrambling_key, 16);
  // END OF SCRAMBLER

  // Lower switch
  FunctionPass *lower = createLowerSwitchPass();
  lower->runOnFunction(*f);

```

在函数开始，使用`LowerSwitchPass`去除 switch，将 switch 结构换成 if 结构。

```
// Nothing to flatten
if (origBB.size() <= 1) {
  return false;
}

```

只有一个基本块的函数将不处理。

```
// Get a pointer on the first BB
Function::iterator tmp = f->begin();  //++tmp;
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

// Remove jump
  insert->getTerminator()->eraseFromParent();

```

如果第一个基本块的末尾是有条件的跳转指令，那么需要将它分割开，并且将它保存到`origBB`（这个数组包含着所有的基本块，除了函数的第一个基本块）。

```
// Create switch variable and set as it
switchVar =
    new AllocaInst(Type::getInt32Ty(f->getContext()), 0, "switchVar", insert);
new StoreInst(
    ConstantInt::get(Type::getInt32Ty(f->getContext()),
                     llvm::cryptoutils->scramble32(0, scrambling_key)),
    switchVar, insert);

```

创建 switch 变量并且初始化。

```
// Create main loop
loopEntry = BasicBlock::Create(f->getContext(), "loopEntry", f, insert);
loopEnd = BasicBlock::Create(f->getContext(), "loopEnd", f, insert);

load = new LoadInst(switchVar, "switchVar", loopEntry);

// Move first BB on top
insert->moveBefore(loopEntry);
BranchInst::Create(loopEntry, insert);

// loopEnd jump to loopEntry
BranchInst::Create(loopEntry, loopEnd);

BasicBlock *swDefault =
    BasicBlock::Create(f->getContext(), "switchDefault", f, loopEnd);
BranchInst::Create(loopEnd, swDefault);

// Create switch instruction itself and set condition
switchI = SwitchInst::Create(&*f->begin(), swDefault, 0, loopEntry);
switchI->setCondition(load);

```

创建两个基本块，存放循环头和尾的指令。然后将`first bb`移到到`loopEntry`的前面，并且创建一条跳转指令，从`first bb`跳到`loopEntry`。紧接着创建了一条从`loopEnd`跳到`loopEntry`的指令。最后，创建了`switch`指令和`switch default`块，并且创建相应的跳转。现在有了大概如下的结构：

```
                  +--------+
                  |first bb|
                  +---+----+
                      |
                 +----v----+
                 |loopEntry| <--------+
                 +----+----+          |
                      |               |
                 +----v---+           |
                 | switch |           |
                 +----+---+           |
                      |               |
       +--------------+               |
       |                              |
+------v-------+                      |
| default case |                      |
+------+-------+                      |
       |                              |
       +-------------+                |
                     |                |
                     |                |
                     v                |
               +-----+------+         |
               |  loopEnd   +---------+
               +------------+

```

```
// Remove branch jump from 1st BB and make a jump to the while
  f->begin()->getTerminator()->eraseFromParent();

  BranchInst::Create(loopEntry, &*f->begin());

```

删除`first bb`的跳转指令，改为跳转到`loopEntry`

```
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

将所有的基本块加入`switch`结构

```
// Recalculate switchVar
for (vector<BasicBlock *>::iterator b = origBB.begin(); b != origBB.end();
     ++b) {
  BasicBlock *i = *b;
  ConstantInt *numCase = NULL;

  // Ret BB
  if (i->getTerminator()->getNumSuccessors() == 0) {
    continue;
  }

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

接下来是根据原先的跳转来计算 switch 变量。对于没有后继（return BB）的基本块，直接跳过。对于只有一个后继的基本块，首先删除跳转指令，并且通过后继基本块来搜索对应的 switch case，根据 case 创建一条存储指令，达到跳转的目的。

```
    // If it's a conditional jump
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
}

```

两个后继的情况跟一个后继的处理方法相似，不同的是，创建一条 select 指令，根据条件的结果来选择分支。

#### [](#虚假控制流 "虚假控制流")虚假控制流

```
-mllvm -bcf: activates the bogus control flow pass
-mllvm -bcf_loop=3: if the pass is activated, applies it 3 times on a function. Default: 1
-mllvm -bcf_prob=40: if the pass is activated, a basic bloc will be obfuscated with a probability of 40%. Default: 30

```

```
void bogus(Function &F) {
    ...

    int NumObfTimes = ObfTimes;

    // Real begining of the pass
    // Loop for the number of time we run the pass on the function
    do{
      DEBUG_WITH_TYPE("cfg", errs() << "bcf: Function " << F.getName()
          <<", before the pass:\n");
      DEBUG_WITH_TYPE("cfg", F.viewCFG());
      // Put all the function's block in a list
      std::list<BasicBlock *> basicBlocks;
      for (Function::iterator i=F.begin();i!=F.end();++i) {
        basicBlocks.push_back(&*i);
      }

```

do 循环的条件表达式是`--NumObfTimes > 0`，`NumObfTimes`关联着`-bcf_loop`选项的值。首先，还是保存基本块。

```
      while(!basicBlocks.empty()){
        NumBasicBlocks ++;
        // Basic Blocks' selection

        if((int)llvm::cryptoutils->get_range(100) <= ObfProbRate){
          DEBUG_WITH_TYPE("opt", errs() << "bcf: Block "
              << NumBasicBlocks <<" selected. \n");
          hasBeenModified = true;
          ++NumModifiedBasicBlocks;
          NumAddedBasicBlocks += 3;
          FinalNumBasicBlocks += 3;
          // Add bogus flow to the given Basic Block (see description)
          BasicBlock *basicBlock = basicBlocks.front();
          addBogusFlow(basicBlock, F);
        }
        else{
          DEBUG_WITH_TYPE("opt", errs() << "bcf: Block "
              << NumBasicBlocks <<" not selected.\n");
        }
        // remove the block from the list
        basicBlocks.pop_front();

        if(firstTime){ // first time we iterate on this function
          ++InitNumBasicBlocks;
          ++FinalNumBasicBlocks;
        }
      } // end of while(!basicBlocks.empty())

      ....
    }while(--NumObfTimes > 0);
}

```

遍历基本块，随机决定当前基本块是否需要修改，`ObfProbRate`变量关联着`-bcf_prob`选项的值。如果命中，则调用`addBogusFlow`函数。

```
virtual void addBogusFlow(BasicBlock * basicBlock, Function &F){
  ...
  BasicBlock::iterator i1 = basicBlock->begin();
  if(basicBlock->getFirstNonPHIOrDbgOrLifetime())
    i1 = (BasicBlock::iterator)basicBlock->getFirstNonPHIOrDbgOrLifetime();
  Twine *var;
  var = new Twine("originalBB");
  BasicBlock *originalBB = basicBlock->splitBasicBlock(i1, *var);

```

在`addBogusFlow`函数中，首先基本块分为两部分，第一部分只包含 PHI 结点、调试信息、lifttime，第二部分包含着剩余的指令。

```
Twine * var3 = new Twine("alteredBB");
BasicBlock *alteredBB = createAlteredBasicBlock(originalBB, *var3, &F);

```

接着，由`createAlteredBasicBlock`函数复制基本块，并增加一些花指令。

```
Twine * var4 = new Twine("condition");
FCmpInst * condition = new FCmpInst(*basicBlock, FCmpInst::FCMP_TRUE , LHS, RHS, *var4);

BranchInst::Create(originalBB, alteredBB, (Value *)condition, basicBlock);

BranchInst::Create(originalBB, alteredBB);

```

现在，我们有三个基本块，第一个是`basicBlock`，`addBogusFlow`函数的参数，第二个是`originalBB`，由`basicBlock`分割出来，第三个是`alteredBB`，由`createAlteredBasicBlock`创建的混淆块，然后将主要是将这三个块拼接起来：

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

```
BasicBlock::iterator i = originalBB->end();

Twine * var5 = new Twine("originalBBpart2");
BasicBlock * originalBBpart2 = originalBB->splitBasicBlock(--i , *var5);

originalBB->getTerminator()->eraseFromParent();

Twine * var6 = new Twine("condition2");
FCmpInst * condition2 = new FCmpInst(*originalBB, CmpInst::FCMP_TRUE , LHS, RHS, *var6);
BranchInst::Create(originalBBpart2, alteredBB, (Value *)condition2, originalBB);

```

在`addBogusFlow`函数的最后，将`originalBB`的最后一条语句分割出来，然后拼接：

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

`addBogusFlow`和`bogus`函数执行完成后，回到`runOnFunction`，接着执行`doF`函数

```
bool doF(Module &M){
     Twine * varX = new Twine("x");
     Twine * varY = new Twine("y");
     Value * x1 =ConstantInt::get(Type::getInt32Ty(M.getContext()), 0, false);
     Value * y1 =ConstantInt::get(Type::getInt32Ty(M.getContext()), 0, false);

     GlobalVariable     * x = new GlobalVariable(M, Type::getInt32Ty(M.getContext()), false,
         GlobalValue::CommonLinkage, (Constant * )x1,
         *varX);
     GlobalVariable     * y = new GlobalVariable(M, Type::getInt32Ty(M.getContext()), false,
         GlobalValue::CommonLinkage, (Constant * )y1,
         *varY);

```

首先，创建了两个全局变量。

```
for(Module::iterator mi = M.begin(), me = M.end(); mi != me; ++mi){
        for(Function::iterator fi = mi->begin(), fe = mi->end(); fi != fe; ++fi){

          TerminatorInst * tbb= fi->getTerminator();

          if(tbb->getOpcode() == Instruction::Br){

            BranchInst * br = (BranchInst *)(tbb);

            if(br->isConditional()){

              FCmpInst * cond = (FCmpInst *)br->getCondition();
              unsigned opcode = cond->getOpcode();

              if(opcode == Instruction::FCmp){
                if (cond->getPredicate() == FCmpInst::FCMP_TRUE){
                  toDelete.push_back(cond); // The condition
                  toEdit.push_back(tbb);    // The branch using the condition
                }
              }
            }
          }
        }
      }

```

紧接着，遍历模块的所有基本块，搜索出条件永远为`true`的比较语句。

```
for(std::vector<Instruction*>::iterator i =toEdit.begin();i!=toEdit.end();++i){
  opX = new LoadInst ((Value *)x, "", (*i));
  opY = new LoadInst ((Value *)y, "", (*i));

  op = BinaryOperator::Create(Instruction::Sub, (Value *)opX,
      ConstantInt::get(Type::getInt32Ty(M.getContext()), 1,
        false), "", (*i));
  op1 = BinaryOperator::Create(Instruction::Mul, (Value *)opX, op, "", (*i));
  op = BinaryOperator::Create(Instruction::URem, op1,
      ConstantInt::get(Type::getInt32Ty(M.getContext()), 2,
        false), "", (*i));
  condition = new ICmpInst((*i), ICmpInst::ICMP_EQ, op,
      ConstantInt::get(Type::getInt32Ty(M.getContext()), 0,
        false));
  condition2 = new ICmpInst((*i), ICmpInst::ICMP_SLT, opY,
      ConstantInt::get(Type::getInt32Ty(M.getContext()), 10,
        false));
  op1 = BinaryOperator::Create(Instruction::Or, (Value *)condition,
      (Value *)condition2, "", (*i));

  BranchInst::Create(((BranchInst*)*i)->getSuccessor(0),
      ((BranchInst*)*i)->getSuccessor(1),(Value *) op1,
      ((BranchInst*)*i)->getParent());
  DEBUG_WITH_TYPE("gen", errs() << "bcf: Erase branch instruction:"
      << *((BranchInst*)*i) << "\n");
  (*i)->eraseFromParent(); // erase the branch
}

```

在最后，用表达式`(x - 1) * x % 2 == 0 || y < 0`替换`永远为true`的比较语句。从上面可以看出，`doF`函数将所有`永远为true`的比较语句替换为`(x - 1) * x % 2 == 0 || y < 0`，并且，这个表达式的结果也`永远为true`

#### [](#指令替换 "指令替换")指令替换

```
-mllvm -sub: activate instructions substitution
-mllvm -sub_loop=3: if the pass is activated, applies it 3 times on a function. Default : 1.

```

```
bool Substitution::substitute(Function *f) {
  Function *tmp = f;

  int times = ObfTimes;
  do {
    for (Function::iterator bb = tmp->begin(); bb != tmp->end(); ++bb) {
      for (BasicBlock::iterator inst = bb->begin(); inst != bb->end(); ++inst) {

        if (inst->isBinaryOp()) {

          switch (inst->getOpcode()) {

          case Instruction::Add:
            (this->*funcAdd[llvm::cryptoutils->get_range(NUMBER_ADD_SUBST)])(
                cast<BinaryOperator>(inst));
            break;

          case BinaryOperator::Sub:
            (this->*funcSub[llvm::cryptoutils->get_range(NUMBER_SUB_SUBST)])(
                cast<BinaryOperator>(inst));
            break;

          case Instruction::And:
            (this->*funcAnd[llvm::cryptoutils->get_range(2)])(cast<BinaryOperator>(inst));
            break;

          case Instruction::Or:
            (this->*funcOr[llvm::cryptoutils->get_range(2)])(cast<BinaryOperator>(inst));
            break;

          case Instruction::Xor:
            (this->*funcXor[llvm::cryptoutils->get_range(2)])(cast<BinaryOperator>(inst));
            break;
          }              // End switch
        }                // End isBinaryOp
      }                  // End for basickblock
    }                    // End for Function
  } while (--times > 0); // for times
  return false;
}

```

从源码可以看出，ollvm 只对加、减、或、与、异或这五种操作进行替换，`funcXXX`变量都是函数数组，随机的选择一种变换进行操作。`ObfTimes`关联`-sub_loop`。

```
// a = b - (-c)
op = BinaryOperator::CreateNeg(bo->getOperand(1), "", bo);
op = BinaryOperator::Create(Instruction::Sub, bo->getOperand(0), op, "", bo);
bo->replaceAllUsesWith(op);

// a = -(-b + (-c))
op = BinaryOperator::CreateNeg(bo->getOperand(0), "", bo);
op2 = BinaryOperator::CreateNeg(bo->getOperand(1), "", bo);
op = BinaryOperator::Create(Instruction::Add, op, op2, "", bo);
op = BinaryOperator::CreateNeg(op, "", bo);
bo->replaceAllUsesWith(op);

// r = rand (); a = b + r; a = a + c; a = a - r
Type *ty = bo->getType();
ConstantInt *co = (ConstantInt *)ConstantInt::get(ty, llvm::cryptoutils->get_uint64_t());
op = BinaryOperator::Create(Instruction::Add, bo->getOperand(0), co, "", bo);
op = BinaryOperator::Create(Instruction::Add, op, bo->getOperand(1), "", bo);
op = BinaryOperator::Create(Instruction::Sub, op, co, "", bo);
bo->replaceAllUsesWith(op);

// r = rand (); a = b - r; a = a + b; a = a + r
Type *ty = bo->getType();
ConstantInt *co = (ConstantInt *)ConstantInt::get(ty, llvm::cryptoutils->get_uint64_t());
op = BinaryOperator::Create(Instruction::Sub, bo->getOperand(0), co, "", bo);
op = BinaryOperator::Create(Instruction::Add, op, bo->getOperand(1), "", bo);
op = BinaryOperator::Create(Instruction::Add, op, co, "", bo);
bo->replaceAllUsesWith(op);

```

上面对应着`funcAdd`数组的四种替换方法。第一种，将第二个操作数取反，然后改写成减法指令。第二种，将两个操作数都取反，结果相加之后再次取反。第三种，取一个随机数，将随机数与操作数 1 相加，然后将结果与操作数 2 相加，最后减去随机数。第四种，取一个随机数，将操作数 1 减去随机数，然后将结果与操作数 2 相加，最后加上随机数。

指令替换的过程都比较简单，也就不再介绍其他的替换方法了，具体可以参考官网：[Instructions Substitution](https://github.com/obfuscator-llvm/obfuscator/wiki/Instructions-Substitution)。

#### [](#字符串加密 "字符串加密")字符串加密

“字符串加密” 这个功能不是 ollvm 的，而是” [孤挺花](https://github.com/GoSSIP-SJTU/Armariris) “这个项目的，不过它是基于 ollvm 的，所以也分析看看。

与上面的 pass 不同，字符串加密的 pass 是继承`ModulePass`的，所以它的入口函数是`runOnModule`。

```
virtual bool runOnModule(Module &M) {
    std::vector<GlobalVariable*> toDelConstGlob;
    std::vector<encVar*> encGlob;

    for (Module::global_iterator gi = M.global_begin(), ge = M.global_end(); gi != ge; ++gi) {
        GlobalVariable* gv = &(*gi);

        std::string::size_type str_idx = gv->getName().str().find(".str.");
        std::string section(gv->getSection());

        if (gv->isConstant() && gv->hasInitializer() && isa<ConstantDataSequential>(gv->getInitializer()) &&
                section != "llvm.metadata" && section.find("__objc_methname") == std::string::npos) {

```

首先遍历模块的所有全局变量，只有这个变量为常量，并且已经初始化的，才进行下一步操作。

```
        GlobalVariable *dynGV = new GlobalVariable(M,
                                      gv->getType()->getElementType(),
                                      !(gv->isConstant()), gv->getLinkage(),
                                      (Constant*) 0, gv->getName(),
                                      (GlobalVariable*) 0,
                                      gv->getThreadLocalMode(),
                                      gv->getType()->getAddressSpace());

        dynGV->setInitializer(gv->getInitializer());

        Constant *initializer = gv->getInitializer();
        ConstantDataSequential *cdata = dyn_cast<ConstantDataSequential>(initializer);
        if (cdata) {
            const char *orig = cdata->getRawDataValues().data();
            unsigned int len = cdata->getNumElements()*cdata->getElementByteSize();

            encVar *cur = new encVar();
            cur->var = dynGV;
            cur->key = llvm::cryptoutils->get_uint8_t();

            char *encr = (char*)orig; 

            for (unsigned i = 0; i != len; ++i) {
                    encr[i] = orig[i]^cur->key;
            }

            dynGV->setInitializer(initializer);

            encGlob.push_back(cur);
        } else {
            dynGV->setInitializer(initializer);
        }
        gv->replaceAllUsesWith(dynGV);
        toDelConstGlob.push_back(gv);

    }
}

```

然后，创建一个全局变量，并且取得一个随机数，将随机数作为 key 与原先原先全局变量的值进行异或，最后，将结果设为新全局变量的值。

```
for (unsigned i = 0, e = toDelConstGlob.size(); i != e; ++i)
        toDelConstGlob[i]->eraseFromParent();

addDecodeFunction(&M, &encGlob);

return true;

```

在这个函数的末尾，删除原先的全局变量，然后调用`addDecodeFunction`函数，增加一个解密函数。

```
void addDecodeFunction(Module *mod, std::vector<encVar*> *gvars) {

    std::vector<Type*>FuncTy_args;
    FunctionType* FuncTy = FunctionType::get(
      Type::getVoidTy(mod->getContext()),  // returning void
      FuncTy_args,  // taking no args
      false);

    uint64_t StringObfDecodeRandomName = cryptoutils->get_uint64_t();
    std::string  random_str;
    std::strstream random_stream;
    random_stream << StringObfDecodeRandomName;
    random_stream >> random_str;
    StringObfDecodeRandomName++;
    Constant* c = mod->getOrInsertFunction(".datadiv_decode" + random_str, FuncTy);
    Function* fdecode = cast<Function>(c);
    fdecode->setCallingConv(CallingConv::C);

    ConstantInt* const_0 = ConstantInt::get(mod->getContext(), APInt(32, 0));
    ConstantInt* const_1 = ConstantInt::get(mod->getContext(), APInt(32, 1));
    BasicBlock* label_entry = BasicBlock::Create(mod->getContext(), "entry", fdecode);

```

开始，创建一个函数声明: `void .datadiv_decodexxx()`，生成了两个常量和一个基本块，这个基本块作为函数的头

```
for (unsigned i = 0, e = gvars->size(); i != e; ++i) {
   GlobalVariable *gvar = (*gvars)[i]->var;
   char key = (*gvars)[i]->key;

   Constant *init = gvar->getInitializer();
   ConstantDataSequential *cdata = dyn_cast<ConstantDataSequential>(init);

   unsigned len = cdata->getNumElements()*cdata->getElementByteSize();

   ConstantInt* const_len = ConstantInt::get(mod->getContext(), APInt(32, len));
   BasicBlock* label_for_body = BasicBlock::Create(mod->getContext(), "for.body", fdecode, 0);
     BasicBlock* label_for_end = BasicBlock::Create(mod->getContext(), "for.end", fdecode, 0);

```

在循环里面，创建了两个基本块，分别作为解密循环的 body 和 end，并且计算出了当前 key 的长度

```
ICmpInst* cmp = new ICmpInst(*label_entry, ICmpInst::ICMP_EQ, const_len, const_0, "cmp");
BranchInst::Create(label_for_end, label_for_body, cmp, label_entry);

```

在这里，插入了第一条比较指令，条件表达式是：`key的长度 == 0`，如果为 0 则跳出循环，否则开始执行循环

```
Argument* fwdref_18 = new Argument(IntegerType::get(mod->getContext(), 32));
PHINode* int32_i = PHINode::Create(IntegerType::get(mod->getContext(), 32), 2, "i.09", label_for_body);
int32_i->addIncoming(fwdref_18, label_for_body);
int32_i->addIncoming(const_0, label_entry);

```

在解密循环的 body 里面，创建了一个 PHI 结点，数据源分别是 0 和解密循环自增变量

```
std::vector<Value*> ptr_32_indices;
ptr_32_indices.push_back(const_0);
ptr_32_indices.push_back(int64_idxprom);

ArrayRef<Value*> ref_ptr_32_indices = ArrayRef<Value*>(ptr_32_indices);
Instruction* ptr_arrayidx = GetElementPtrInst::Create(NULL, gvar, ref_ptr_32_indices, "arrayidx", label_for_body);
LoadInst* int8_20 = new LoadInst(ptr_arrayidx, "", false, label_for_body);
int8_20->setAlignment(1);

```

这里生成的指令作用是：使用解密循环自增变量作为数组索引，从加密后的字符串里面读取一个字节

```
ConstantInt* const_key = ConstantInt::get(mod->getContext(), APInt(8, key));
BinaryOperator* int8_dec = BinaryOperator::Create(Instruction::Xor, int8_20, const_key, "xor", label_for_body);
StoreInst* void_21 = new StoreInst(int8_dec, ptr_arrayidx, false, label_for_body);
void_21->setAlignment(1);

```

生成一条异或指令，将 key 和数据进行异或，得到原数据，最后将它写回。

```
        BinaryOperator* int32_inc = BinaryOperator::Create(Instruction::Add, int32_i, const_1, "inc", label_for_body);

        ICmpInst* int1_cmp = new ICmpInst(*label_for_body, ICmpInst::ICMP_EQ, int32_inc, const_len, "cmp");
        BranchInst::Create(label_for_end, label_for_body, int1_cmp, label_for_body);

        fwdref_18->replaceAllUsesWith(int32_inc); delete fwdref_18;

        label_entry = label_for_end;
    }
ReturnInst::Create(mod->getContext(), label_entry);

```

在最后，将解密循环自增变量自增，并且创建比较指令，当解密没完成时回到循环体继续解密。在接下来的代码，主要是获取 [llvm](https://so.csdn.net/so/search?q=llvm&spm=1001.2101.3001.7020) 的全局变量`llvm.global_ctors`，然后将这个解密函数插入`global_ctors`，达到在运行时解密的功能。