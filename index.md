## Here we are Orcs

Hi, I’m Praveen Velliengiri, this is a write-up of my Google Summer of Code 2019 with LLVM Compiler Infrastructure. LLVM is a modern compiler toolchain, checkout [here](https://llvm.org/). 
Find me @ [![alt text][1.1]][1] [![alt text][2.1]][2]

[1.1]: http://i.imgur.com/wWzX9uB.png (twitter icon with padding)
[2.1]: http://i.imgur.com/9I6NRUm.png (github icon with padding)
[1]: https://twitter.com/PreeJackie
[2]: http://www.github.com/preejackie
### Contents-
- [LLVM Community](#llvm-community)
- [What is ORC?](#what-is-orc)
- [Speculative Compilation](#speculative-compilation)
   - [High Level Design](#high-level-design)
   - [Talk less, show me code!](#talk-less-show-me-code)
   - [How IR is instrumented?](#how-ir-is-instrumented)
   - [What is the Speed Up?](#what-is-the-speed-up)
   - [Current Status](#current-status)
- [What's Next?](#whats-next)
- [What I learn?](#what-i-learn)

### LLVM Community
LLVM is not a million lines of code, but it is a group of people who make it possible :) This in general applies to all Open Source Orgs. Over time I find the culture in LLVM is extremely friendly (mainly for newcomers). LLVM developers are very supportive and kind enough to explain whatever complex LLVM concepts. I used to ask very stupid questions like (hey, how to traverse a Basic Block) and still does :)  Basically, they are just a bunch of cool folks! I highly recommend anyone to be part of this awesome community. Seriously, take my words for it.

This wouldn’t have happened without my 2 mentors - Lang Hames & David Blaikie. Over this course of time, they gave me incredible support and helped me to understand LLVM, development process, how ORC works, etc. They both work on LLVM as their day job, Cool right?
    They replied to all of my emails, chats & questions :) I learn a ton of great stuff from them. Honestly, I’m very lucky to work with them. They ultimately became my role model.

### What is ORC?
In LLVM, we already have ELF (Executable & Linking Format) and DWARF(Debugging With Attributed Record Formats) so include bad guys also - ORC's, just kidding :) 
ORC(On Request Compilation) is a fairly new LLVM Concurrent JIT Infrastructure, it is not very tightly coupled with LLVM though. You can write your own compilers for your own program representations (like AST, IR, or anything you call) and you can freely use components of ORC. To understand ORC you have to think from the Linker’s Perspective, that is how I can link the symbols, how I can resolve and relocate symbols? 

You can add your program representations to JIT and compilers that know how to reduce your program representation to machine code. ORC know when and how to run your compilers for you, which I call the ORC’s spell :) With the help of orc’s spell you can add multiple compilers and multiple program representations to your JIT, which eventually turns your JIT into a Compiler orchestration system, Isn’t exciting? 

ORC triggers the compilation of symbols (data, function) when it is looked up via `ExecutionSession::lookup`. It also manages concurrent compilation for you: multiple compilers can run at once, and multiple concurrent lookups can be made for symbols.

It also supports lazy compilation - that is you compile when you need it. It is an important feature  to have in Just in time Compilation Engine. It helps the Apps to start faster without paying ahead of time compilation cost, but as you imagine you can’t run your program until it is converted into binary, there is a cost to compilation during the execution of App.

At first, I knew nothing about ORC. I have to understand them to work in them. This was tough because I didn’t have any experience with JIT or Linkers before. There is no detailed doc at that time, so I choose to read the entire project & add `llvm::errs` to understand what is really going under the hood. It was tricky at first, but eventually, I understand the Orc’s spell. It was rewarding, really!! 
    
The primary aim is to develop a proof-of-concept of speculative compilation to understand how it works, whether we can improve & solidify in future LLVM release for all to use. Now that we know about ORC we can jump into my project, Are you ready?

## Speculative Compilation:
This is the project I spend my summer on! The main aim of this project is to hide compilation time which is mingled with the App execution time. Okay, but how to do that? 
    Orc supports concurrent compilation - the key idea here is to use this feature to lookup symbols early before they even get executed. By using the additional core’s you can reduce your total execution time :) But how do we choose what to compile early? That’s the key ingredient of this entire work, speculating intelligently, that is knowing which function will likely be called next.
    
### Key Goals
- Infrastructure to support speculation.
- Building LLVM IR analysis queries to select likely functions.
- Writing custom JIT stacks.
- Benchmarking our prototype


### High Level Design
ORC supports orthogonal feature set, so you can mix and match to build your custom JIT stack and play around it. But Speculative Compilation is non-orthogonal, it is bounded with Lazy compilation. ORC has layer concept which processes your program representation and emits the result to the layer below - examples of a layer in the trunk are LLVM IR transformation layer, Compiler layer, Object Layer

Compilation flow:
    IR transformation Layer -> Compiler Layer -> Object Layer. 

For our speculation to work we have to communicate between those layers: 
We introduce an **_ImplSymbolMap_** class - this helps us to track the Implementation symbol and Impl JITDylib of facade symbols. This is the duplicate information we keep around and helps us to move forward with our proof-of-concept goal. We have decided to remove this information by sharing the single copy of this information between lazy and lazy+speculative compilation scheme. 

**_Speculator_**:
    This is an important class, it serves two responsibilities to us:
Register which symbols are likely for the given symbol.
Launch speculative compilation for likely symbols, when entered from the JIT’d code. 

**_IRSpeculationLayer_**:
    It follows a layer concept, it instruments the LLVM IR program representation with globals, and runtime calls into the JIT compiler which helps to jump back into ORC during application execution to launching compiles and return. 

**_SpeculateQuery_**:
    They are normally function objects, which query the given unit of IR (function) and return the which are all the likely next executable symbols. 
   
   1. **_BlockFreqQuery_** : This heuristic computes the static block frequencies of function’s basic blocks. The blocks with higher frequencies tend to execute commonly. So the idea here is to get the blocks with high frequencies and extract the target functions of the LLVM CallInst in that basic block.

But we observe that those blocks often occur inside a loop or that may be a block with many predecessors, this means that are still essentially not covering blocks with calls that are nearer to the Entry Basic Block. 

   2. **_SequenceBBQuery_** : This heuristic tries hard to find the actual sequence of basic blocks that occurs in a valid path from entry block of control flow graph (CFG) to the exit block of CFG. With this heuristic, we get the set of basic block in the order of their execution in all possible CFG path’s which have hot basic block’s.

### Talk less, show me code!

Since this is a new feature, we like to introduce it as a big changeset by following WIP commit style. 

- [Speculative Compilation](https://reviews.llvm.org/D63378) : This is the main patch that introduces the infrastructure needed for speculation in LLVM ORCv2, this revision includes : establishing ImplSymbolMap to track symbol location between layers, Speculator with __orc_speculator target function, preliminary IR instrumentation with no separate function global, block frequency heuristic, add custom JIT Stack.

- [New Speculate Query Implementation](https://reviews.llvm.org/D66399) : This Patch includes - Following Thread Safety Contract, Implements SequenceBB Query speculation heuristic, Implement new IR instrumentation with Per function globals and guards.

- [Assertion race](https://reviews.llvm.org/D63377) : Avoid race conditions in debug mode.

- [Kaleidoscope update](https://reviews.llvm.org/D62491) : Update kaleidoscope tutorial chapter 3 to follow ORC version 2 JIT APIs.

- [Unique JD names](https://reviews.llvm.org/D62139) : Ensuring unique names for facade JITDylib's for better understanding while debugging.

- [D66289](https://reviews.llvm.org/D66289) : Remove unimplemented query.

All the revisions mentioned above got accepted,
Merged Revisions:
- [Speculative Compilation](https://reviews.llvm.org/D63378) is committed through [f5c40cb9002a](https://reviews.llvm.org/rGf5c40cb9002a7cbddec66dc4b440525ae1f14751).
- [New Speculate Query Implementation](https://reviews.llvm.org/D66399) is committed through [rL370092](https://reviews.llvm.org/rL370092).
- [D66289](https://reviews.llvm.org/D66289) is committed through [rL370085](https://reviews.llvm.org/rL370085).
- [Unique JD names](https://reviews.llvm.org/D62139) is committed through [rL361215](https://reviews.llvm.org/rL361215).

### How IR is instrumented?
There are two ways to make speculative compilation work:
1. Through Instrumentation
2. Embedding a separate entity in Orc itself to perform speculation.

 For a proof-of-concept, as things currently stand we perform speculation through LLVM IR Instrumentation.
 
 LLVM IR before instrumentation:
 
 ```llvm
declare dso_local void @foo()
declare dso_local void @bar()

define dso_local i32 @main() { 
entry:
  ; omitted for beverity
  %4 = icmp ne i32 %3, 0
  br i1 %4, label %Ontrueblock, label %Onfalseblock
Ontrueblock:                                                
  call void @foo()
  br label %exit
Onfalseblock:                                           
  call void @bar()
  br label %exit
exit:
  ret i32 0
}
 ```
Here the main function contains 4 basic blocks, and based on the branch instruction: `%4 = icmp ne i32 %3, 0`. Control jumps either to Ontrueblock or Onfalseblock and calls either foo or bar respectively.

LLVM IR after instrumentation: 

```llvm
%Class.Speculator = type opaque
@__orc_speculator = external global %Class.Speculator
@__orc_speculate.guard.for.main = internal local_unnamed_addr global i8 0, align 1

declare dso_local void @foo() 
declare dso_local void @bar() 
declare void @__orc_speculate_for(%Class.Speculator* %0, i64 %1)

define dso_local i32 @main(){
__orc_speculate.decision.block:
  %guard.value = load i8, i8* @__orc_speculate.guard.for.main
  %compare.to.speculate = icmp eq i8 %guard.value, 0
  br i1 %compare.to.speculate, label %__orc_speculate.block, label %program.entry

__orc_speculate.block:                    
  call void @__orc_speculate_for(%Class.Speculator* @__orc_speculator, i64 ptrtoint (i32 ()* @main to i64))
  store i8 1, i8* @__orc_speculate.guard.for.main
  br label %program.entry

program.entry:                                               
  ; omitted for beverity
  %4 = icmp ne i32 %3, 0
  br i1 %4, label %Ontrueblock, label %Onfalseblock
Ontrueblock:                                               
  call void @foo()
  br label %exit
Onfalseblock:                                            
  call void @bar()
  br label %exit
exit:                                            
  ret i32 0
}

```
Instrumentation code emit declarations, definitions of global and add two new basic block’s namely `__orc_speculate.decision.block` and `__orc_speculate.block` which mutate the CFG structure of @main function. 

In a nutshell, for each function which calls other functions, the IRSpeculationLayer::emit method create a guard.value which is global i8 with internal linkage and initialize with 0. Upon the execution of `@main`, we check whether we executed the function before or not, if the function is not executed yet, the control jumps to `__orc.speculate.block` and call `__orc_speculate_for`, passing the function’s own compiled address, this jumps back into the Orc and launch speculative compiles of very likely functions of function `@main`.

The compare instruction in `__orc_speculate.decision.block` : `br i1 %compare.to.speculate, label %__orc_speculate.block, label %program.entry` guard us from re-entering into JIT on the second invocation of function.

### What is the Speed Up?
In the end, many people wanted to know: did we get any speedup or improvements in general? Especially in compilers, people love this word - performance. If you are one of them, then this is the section for you. 
We have seen consistent speedup in all applications with our proof-of-concept “Speculation” model. 

Here we compare Laziness + Speculation configuration with Orc Lazy Compilation configuration.

![spec403.png](https://raw.githubusercontent.com/preejackie/GSoC-2019-LLVM/master/spec403.png)

These results are for the SPEC 403.gcc benchmark program, we see the speedup of  > 40% over Lazy compilation counterpart, with 10 dedicated compile threads we reduce the total execution time of application (wall clock time) from 17.4 seconds to 9.6 seconds.

![403.gcc.wait.time](https://raw.githubusercontent.com/preejackie/GSoC-2019-LLVM/master/specwait.png)

The Y-axis represents the total time spent in symbol lookup, (amount of time execution thread wait to get runnable code for functions). As we see from the graph, by having at least two dedicated compile threads we can reduce the total wait time over 35%.

Our proof-of-concept technique is also performing well in lightweight apps:

**Execution Time** 

Lazy Scheme | Speculative Scheme
------------ | -------------
pathfinder - 4.921s | 1.672s
tinyexpr   - 5.000s | 4.100s
dbms app   - 5.447s | 4.323s
bufferapp  - 2.000s | 1.324s

We conclude that by solidifying our infrastructure we can still squeeze maximum performance from the machine. 

### Current Status

You can check out the LLVM and you can use the work! The committed code is self-contained, we have also added an example custom JIT Stack with Speculation scheme we hope it will serve as an example for people to update and configure their JIT stacks. You can find the example in main LLVM trunk - [SpeculativeJIT](https://reviews.llvm.org/source/llvm-github/browse/master/llvm/examples/SpeculativeJIT/)

### What's Next?

1. I tried hard to include the LLVM dynamic profiling data into ORC, to effectively improve our heuristics, but I didn't figured how to integrate the profile sections in an object (ELF) with ORC. We definitely include the profiling part to ORC, once we figured out how to use profile sections.

LLVM ORC is fairly new, it needs more features and efficient data structures & algorithms to gain more performance.

### What I learn?
1. I learned that I love to code, more than I thought it before. 
2. I learn more about LLVM, JIT, Linkers, Object Files :)
3. Most importantly, I learned that I don't know many stuff's which is cool, because now I know that - it's time to improve.

### See you!
Thanks for checking this out, if you any queries regarding GSoC always free to get in touch with me `Twitter handle - @preejackie`
