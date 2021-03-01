---
title:  "iOS 编译、链接总结"
date:   2020-12-19 22:11:36 +0530
author: taoye
categories: [iOS, 2020年复现总结]
tags: [iOS]
---

### 流程
![8bb49db8fb82c7eb55cf15b2b48ec3ad](/assets/img/build-link/1D7132F7-AE93-4673-A855-976A117FE96B.jpg)


### 编译过程

1.写入辅助文件：将项目的文件结构对应表、将要执行的脚本、项目依赖库的文件结构对应表写成文件，方便后面使用；并且创建一个 .app 包，后面编译后的文件都会被放入包中；
2.运行预设脚本：Cocoapods 会预设一些脚本，当然你也可以自己预设一些脚本来运行。这些脚本都在 Build Phases 中可以看到；
3.编译文件：针对每一个文件进行编译，生成可执行文件 Mach-O，这过程 LLVM 的完整流程，前端、优化器、后端；
4.链接文件：将项目中的多个可执行文件合并成一个文件；
5.拷贝资源文件：将项目中的资源文件拷贝到目标包；
6.编译 storyboard 文件：storyboard 文件也是会被编译的；
7.链接 storyboard 文件：将编译后的 storyboard 文件链接成一个文件；
8.编译 Asset 文件：我们的图片如果使用 Assets.xcassets 来管理图片，那么这些图片将会被编译成机器码，除了 icon 和 launchImage；
9.运行 Cocoapods 脚本：将在编译项目之前已经编译好的依赖库和相关资源拷贝到包中。
10.生成 .app 包
11.将 Swift 标准库拷贝到包中
12.对包进行签名
13.完成打包


### 详解
* 预编译 （#就是预编译表示）
    - import
    - 内联函数
    - macro宏
    - 其他预处理指令（包括条件编译 if DEBUG这种）
* exical Analysis - 词法分析（输出token流）
    使用 clang 命令 clang -Xclang -dump-tokens main.m，
    词法分析，只需要将源代码以字符文本的形式转化成Token流的形式，不涉及交验语义，不需要递归，是线性的。
    词法分析其实是编译器开始工作真正意义上的第一个步骤，其所做的工作主要为将输入的代码转换为一系列符合特定语言的词法单元，这些词法单元类型包括了关键字，操作符，变量等等。
    什么是token流呢？可以这么理解：就是有"类型"，有"值"的一些小单元。
* Semantic Analysis - 语法分析（输出(AST)抽象语法树）
    - 使用 clang 命令 clang -Xclang -ast-dump -fsyntax-only main.m，转化后的树如下
    ![e99aebc4e7b6b6d761b2bb01dafcb4a9](/assets/img/build-link/EA30C062-18A2-4391-B2F6-1BB796C156C2.png)
* 语义分析
* 静态分析
    - 通过语法树进行代码静态分析，找出非语法性错误
    - 模拟代码执行路径，分析出control-flow graph(CFG) 【MRC时代会分析出引用计数的错误】
    - 预置了常用Checker（检查器）
    - 一旦编译器把源码生成了抽象语法树，编译器可以对这棵树做分析处理，以找出代码中的错误，比如类型检查：即检查程序中是否有类型错误。例如：如果代码中给某个对象发送了一个消息，编译器会检查这个对象是否实现了这个消息（函数、方法）。此外，clang 对整个程序还做了其它更高级的一些分析，以确保程序没有错误。
    - 一般会把类型检查分为两类：动态的和静态的。动态的在运行时做检查，静态的在编译时做检查。以往，编写代码时可以向任意对象发送任何消息，在运行时，才会检查对象是否能够响应这些消息。由于只是在运行时做此类检查，所以叫做动态类型
    - 当在代码中使用 ARC 时，编译器在编译期间，会做许多的类型检查：因为编译器需要知道哪个对象该如何使用。
*  CodeGen - （Intermediate Representation，简称IR）IR中间代码生成
    - 编译器后端主要包括代码生成器、代码优化器。代码生成器将中间代码转换为目标代码，代码优化器主要是进行一些优化，比如删除多余指令，选择合适寻址方式等，如果开启了 bitcode 苹果会做进一步的优化，有新的后端架构还是可以用这份优化过的 bitcode 去生成。优化中间代码生成输出汇编代码，把之前的 .i 文件转换为汇编语言，产生 .s 文件
    - 使用命令 clang -S -emit-llvm main3.m -o main.ll生成LLVM 中间代码LLVM IR
    - LLVM IR 是Frontend的输出，也是LLVM Backend的输入，前后端的桥接语言, 更具生成的文件解析，其实生成的LLVM IR对Runtime进行桥接的一个文件
        1.Class/Meta Class/Protocol/Category内存结构生成，并存放在指定section中（如Class：_DATA,_objc_classrefs）
        2.Method/lvar/Property内存结构生成
        3.组成method_list/ivar_list/property_list并填入Class
        4.Non-Fragile ABI:为每个Ivar合成OBJC_IVAR_$_ 偏移值常量
        5.存取Ivar的语句（ivar = 123; int a = ivar;）转写成base + OBJC_IVAR$_的形式
        6.将语法树中的ObjcMessageExpr翻译成相应版本的objc_msgSend，
        7.对super关键字的调用翻译成objc_msgSendSuper
        8.根据修饰符strong/weak/copy/atomic合成@property 自动实现的 setter/getter
        9.处理@synthesize
        10.生成block_layout的数据结构
        11.变量的capture(__block/__weak)
        12.生成_block_invoke函数
        13.ARC：分析对象引用关系，将objc_storeStrong/objc_storeWeak等ARC代码插入
        14.将ObjCAutoreleasePoolStmt转译成objc_autoreleasePoolPush/Pop
        15.实现自动调用[super dealloc]
        16.为每个拥有ivar的Class合成.cxx_destructor方法来自动释放类的成员变量，代替MRC时代的“self.xxx = nil”
    - 其他分析
        - ObjCUnusedIVarsChecker.cpp是用来检查是否有定义了，但是从未使用过的变量。
        - ObjCSelfInitChecker.cpp是检查在 你的初始化方法中中调用 self 之前，是否已经调用 [self initWith...] 或 [super init] 了。
        ![b87bdbc08ffb7469d14f55719033c3c8](/assets/img/build-link/4A40BA22-D35C-47B3-AE47-A0B3757D8E2E.png)
* Optimize - 优化IR
    - 这一步骤的优化是非常重要的，很多直接转换来的代码是不合适且消耗内存的，因为是直接转换，所以必然会有这样的问题，而优化放在这一步的好处在于前端不需要考虑任何优化过程，减少了前端的开发工作。
* 生成Target相关汇编
    - 使用命令 clang -S -o - main3.m | open -f可以查看生成的汇编代码：
* Link生成Executable
    - 在最后，LLVM 将会把这些汇编代码输出成二进制的可执行文件，使用命令 clang main3.m -o main.out 即可查看
    - 在编译链接期间发挥作用，把目标文件和静态库一起链接形成可执行文件。

    
    


### LLVM
LLVM采用三相设计，前端Clang负责解析，验证和诊断输入代码中的错误，然后将解析的代码转换为LLVM IR，后端LLVM编译把IR通过一系列改进代码的分析和优化过程提供，然后被发送到代码生成器以生成本机机器代码。

LLVM是一个模块化和可重用的编译器和工具链技术的集合，Clang 是 LLVM 的子项目，是 C，C++ 和 Objective-C 编译器，目的是提供惊人的快速编译，比 GCC 快3倍，其中的 clang static analyzer 主要是进行语法分析，语义分析和生成中间代码，当然这个过程会对代码进行检查，出错的和需要警告的会标注出来。LLVM 核心库提供一个优化器，对流行的 CPU 做代码生成支持。lld 是 Clang / LLVM 的内置链接器，clang 必须调用链接器来产生可执行文件。


Clang是LLVM的一个前端，在Xcode编译iOS项目的时候，都是使用的LLVM，其实在编写代码以及调试的时候都在接触LLVM提供的功能，例如：代码的亮度（Clang）、实时代码检查（Clang）、代码提示（Clang）、debug断点调试（LLDB）。


![6691a0ce9b20056e606fc2beff12e777](/assets/img/build-link/00A3FC24-3025-47A8-ABE2-CCD53DB2F073.png)



### Swift的编译过程
在 Swift 编译器结构 的官方文档中描述了 Swift 编译器是如何工作的，分为如下步骤：

解析：解析器是一个简单的递归下降解析器（在 lib / Parse 中实现），带有集成的手动编码词法分析器。解析器负责生成没有任何语义或类型信息的抽象语法树（AST），并针对输入源的语法问题发出警告或错误。
语意分析：语义分析（在 lib / Sema 中实现）负责解析 AST 并将其转换为格式良好的完全检查形式的 AST，并在源代码中发出语义问题的警告或错误。语义分析包括类型推断，如果成功，则所得到的代码是类型检查安全的 AST 。
Clang导入器：Clang导入器（在 lib / ClangImporter 中实现）导入Clang模块，并将它们导出的 C 或 Objective-C API 映射到相应的 Swift API中。结果导入的 AST 可以通过语义分析来引用。
SIL生成：Swift中间语言（Swift Intermediate Language，简称SIL）是一种高级的，Swift特有的中间语言，适用于 Swift 代码的进一步分析和优化。SIL 生成阶段（在 lib / SILGen 中实现）将类型检查的 AST 降低到所谓的 “原始” SIL。SIL的设计描述在 docs/ SIL.rst 中可以看到。
SIL优化：在SIL优化（在 lib/Analysis，lib/ ARC，lib/LoopTransforms，和 lib/Transforms 中实现）执行额外的高级别，Swift 特有的优化的程序，包括（例如）自动引用计数优化，虚拟化和通用专业化。
LLVM IR生成：IR生成（在 lib/IRGen 中实现）将 SIL 降到 LLVM IR，此时LLVM可以继续对其进行优化并生成机器码。

### OC swift 编译区别
iOS 开发中 Objective-C 是 Clang / LLVM 来编译的。

swift 是 Swift / LLVM，其中 Swift 前端会多出 SIL optimizer，它会把 .swift 生成的中间代码 .sil 属于 High-Level IR， 因为 swift 在编译时就完成了方法绑定直接通过地址调用属于强类型语言，方法调用不再是像OC那样的消息发送，这样编译就可以获得更多的信息用在后面的后端优化上。