## Welcome to GitHub Pages

Hi I’m Praveen, this is a write-up of my Google Summer of Code 2019 with LLVM Compiler Infrastructure. I hope many of you know what LLVM is if not, there are a plethora of content available over the web, check it out! 

### LLVM Culture
LLVM is not a million lines of code, but it is a group of people who make it possible :) This applies to all Open Source Orgs. During the course of time I find the culture in LLVM is extremely friendly (mainly for newcomers). LLVM developers are very friendly and kind enough to explain whatever LLVM concepts. I used to ask very stupid questions like (hey, how to traverse a Basic Block) and still does :)  Basically, they are just a bunch of cool folks! I highly recommend anyone to be part of this awesome community. Seriously, you can take my words.

### What is ORC?
It is a fairly new LLVM Concurrent JIT Infrastructure, it is not very tightly coupled with LLVM though. You can write your own compilers for your own program representations (like AST, IR, or anything you call) and you can freely use components of ORC. To understand ORC you have to think from the Linker’s Perspective, that is how I can link the symbols, how I can resolve and relocate symbols? 

You can add your program representations to JIT and compilers that know how to reduce your program representation to machine code. ORC know when and how to run your compilers for you, which I call the orc’s spell :) With the help of orc’s spell you can add multiple compilers and multiple program representations to your JIT, which eventually turns your JIT into a Compiler orchestration system, Isn’t exciting? 

ORC triggers the compilation of symbols (data,function) when it is looked up via (ExecutionSession::lookup). It also manages concurrent compilation for you: multiple compilers can run at once, and multiple concurrent lookups can be made for symbols.

It also supports lazy compilation - that is you compile when you need it. It is an important feature  to have in Just in time Compilation Engine. It helps the Apps to start faster without paying ahead of time compilation cost , but as you imagine you can’t run your program until it is converted into binary, there is a cost to compilation during the execution of App. Oh, by knowing about ORC we eventually jump into my project. Great, right?

### Speculative Compilation:
This is a project I spend my summer on!. The main aim of this project is to hide compilation time which is mingled with the App execution time. Okay, but how to do that? 
    Orc supports concurrent compilation - key idea here is to use this feature to lookup symbols early before they even get executed. By using the additional core’s you can reduce your total execution time :) But randomly speculating should hurt your performance, but making good choices during speculation should improve your performance.
    That’s the key ingredient of this entire work, speculating intelligently, that is knowing which function will likely be called next. 

### Time Travel
Travel back in time with me to the start of this project :) I was preparing for GSoC with LLVM before GSoC program officially starts. I’m eyeing on every mail in LLVM dev listing for key words like GSoC, projects etc. But unfortunately, I didn’t find any projects that are listed a good suit for me. Even I tried to work on two projects to write proposal, but eventually I lost interest in those two also. There only two weeks to write a proposal but I still haven't figured out what I’m going to do. But I have a habit of just watching LLVM dev meetings talks :). Eventually, I came across - LLVM next generation JIT APIs - keynote talk. It feels interesting to me, so I contacted the speaker over the mail. And it happens!

### My Yoda's
This wouldn’t have happened without my 2 mentors - Lang Hames & David Blaikie. Over this course of time, they gave me an incredible support and helped me to understand LLVM, development process, how ORC works, etc. They both work on LLVM as their day job, Cool right?
    They replied to all of my mails, chats & questions :) I learn a ton of great stuff from them. Honestly, I’m very lucky to work with them. They ultimately became my role model.

### My Path 
At first, I knew nothing about ORC. I have to understand them to work in them. This was tough, because I didn’t have any experience with JIT or Linkers before. There is no detailed doc at that time, so I choose to read the entire project & add llvm::errs to understand what is really going under the hood. It was tricky at first, but eventually I understand the Orc’s spell. It was rewarding, really!! 
    
The primary aim is to develop a proof-of-concept of speculative compilation to understand how it works, whether we can improve & solidify in future llvm release for all to use.

### Contributions
ORC supports orthogonal feature set, so you can mix and match to build your custom JIT stack and play around it. But Speculative Compilation is non-orthogonal, it is bounded with Lazy compilation. ORC have layer concept which process your program representation and emit the result to the layer below - examples of layer in trunk are LLVM IR transformation layer, Compiler layer, Object Layer

Compilation flow:
    IR transformation Layer -> Compiler Layer -> Object Layer. 

Inorder for our speculation to work we have to communicate between those layers: 
We introduce a **_ImplSymbolMap_** class - this helps us to track the Implementation symbol and Impl JITDylib of facade symbols. This is the duplicate information we keep around and helps us to move forward with our proof-of-concept goal. We have decided to remove this information by sharing the single copy of this information between lazy and lazy+speculative compilation scheme. 

**_Speculator_**:
    This is an important class, it serves two responsibilities to us:
Register which symbols are likely for the given symbol.
Launch speculative compilation for likely symbols, when entered from the JIT’d code. 

**_IRSpeculationLayer_**:
    It follows a layer concept, it instruments the LLVM IR program representation with globals, and runtime calls into the JIT compiler which helps to jump back into ORC during application execution to launching compiles and return. 

**_SpeculateQuery_**:
    They are normally function objects, which query the given unit of IR (function) and return the which are all the likely next executable symbols. 
   
   1. **_BlockFreqQuery_** : This heuristic computes the static block frequencies of function’s basic blocks. The blocks with higher frequencies tend to execute commonly. So the idea here is to get the blocks with high frequencies and extract the target functions of the llvm CallInst in that basic block.

But we observe that those blocks often occur inside a loop or that may be block with many predecessors, this means that are still essentially not covering blocks with calls that are nearer to the Entry Basic Block. 

   2. **_SequenceBBQuery_** : This heuristic tries hard to find the actual sequence of basic blocks that occurs in a valid path from entry block of control flow graph (CFG) to the exit block of CFG. With this heuristic, we get the set of basic block in the order of their execution in all possible CFG path’s which have hot basic block’s.

### Talk less, show me code!
    Since this is a new feature, we like to introduce it as a big changeset by following WIP commit style. 
Introduce Speculative Compilation [D63378](https://reviews.llvm.org/D63378) in Orc.
SequenceBB query implementation [D66399](https://reviews.llvm.org/D66399). 
Avoid race conditions in debug mode [D63377](https://reviews.llvm.org/D63377).
Update kaleidoscope tutorial [D62491](https://reviews.llvm.org/D62491).
Ensuring unique names for JITDylib [D62139](https://reviews.llvm.org/D62139).
Remove unimplemented query [D66289](https://reviews.llvm.org/D66289)

All the revisions mentioned above got accepted, and [D63378](https://reviews.llvm.org/D63378) is committed through [f5c40cb9002a](https://reviews.llvm.org/rGf5c40cb9002a7cbddec66dc4b440525ae1f14751). When you read this, all the revisions will be committed in llvm trunk. 


```markdown
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
  call void @_Z1gv()
  br label %exit
exit:                                            
  ret i32 0
}
```


```markdown
Syntax highlighted code block

# Header 1
## Header 2
### Header 3

- Bulleted
- List

1. Numbered
2. List

**Bold** and _Italic_ and `Code` text

[Link](url) and ![Image](src)
```

For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/preejackie/GSoC-2019-LLVM/settings). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://help.github.com/categories/github-pages-basics/) or [contact support](https://github.com/contact) and we’ll help you sort it out.
