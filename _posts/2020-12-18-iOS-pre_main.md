---
title:  "iOS pre_main总结"
date:   2020-12-18 22:11:36 +0530
author: taoye
categories: [iOS, 2020年复现总结]
tags: [iOS]
---

### pre_main 预览
以下截图摘取自大牛博客：

![9094](/assets/img/pre-main/6C481B1D-10AE-4FA0-96B2-C3292B8B0BF9.png)
    
![fadfk444](/assets/img/pre-main/1800285E-85DE-4DCD-B6B3-B799B6523C47.png)
    
![444a0f0d2](/assets/img/pre-main/C942B34A-6ED5-4D7A-A5DA-49BB70CFFDCF.png)
    
    
### app pre_main加载流程
    
* 加载 动态库是按照依赖加载的，会加载function，coreFounction,lib_system(), lib_runtime,等
* rebase、bind，主要是做dylib之间的绑定、哈希校验、指针纠错（指向__DATA segment中发生变化，导致后续的指针全部变化,bind，因为oc的动态性，（比如一个dylib中写了另一个dylib里类的category）,bind的时间复杂度会很高，定位内部、外部指针引用，例如字符串、函数等,会有大量的io操作（操作指针
* Rebasing：在镜像内部调整指针的指向，针对mach-o在加载到内存中不是固定的首地址（ASLR）这一现象做数据修正的过程； 
    * 由于ASLR(address space layout randomization)的存在，可执行文件和动态链接库在虚拟内存中的加载地址每次启动都不固定，所以需要这2步来修复镜像中的资源指针，来指向正确的地址。 rebase修复的是指向当前镜像内部的资源指针； 而bind指向的是镜像外部的资源指针。
    * iOS4.3后引入了 ASLR ，dylib会被加载到随机地址，这个随机的地址跟代码和数据指向的旧地址会有偏差，dyld 需要修正这个偏差，做法就是将 dylib 内部的指针地址都加上这个偏移量，偏移量的计算方法如下：
    * Slide = actual_address - preferred_address
    * 然后就是重复不断地对 __DATA 段中需要 rebase 的指针加上这个偏移量。这就又涉及到 page fault 和 COW。这可能会产生 I/O 瓶颈，但因为 rebase 的顺序是按地址排列的，所以从内核的角度来看这是个有次序的任务，它会预先读入数据，减少 I/O 消耗。
    * 在 Rebasing 和 Binding 前会判断是否已经 Prebinding。如果已经进行过预绑定（Prebinding），那就不需要 Rebasing 和 Binding 这些 Fix-up 流程了，因为已经在预先绑定的地址加载好了(iOS14优化了此过程，系统的库都已经做了prebinding，并且优化了方法存储)
    * rebase步骤先进行，需要把镜像读入内存，并以page为单位进行加密验证，保证不会被篡改，所以这一步的瓶颈在IO。bind在其后进行，由于要查询符号表，来指向跨镜像的资源，加上在rebase阶段，镜像已被读入和加密验证，所以这一步的瓶颈在于CPU计算。
    * 
* Binding：
    * 将指针指向镜像外部的内容，binding就是将这个二进制调用的外部符号进行绑定的过程。比如我们objc代码中需要使用到NSObject, 即符号OBJC_CLASS$_NSObject，但是这个符号又不在我们的二进制中，在系统库 Foundation.framework中，因此就需要binding这个操作将对应关系绑定到一起；
    * lazyBinding就是在加载动态库的时候不会立即binding, 当时当第一次调用这个方法的时候再实施binding。 做到的方法也很简单： 通过dyld_stub_binder这个符号来做。lazyBinding的方法第一次会调用到dyld_stub_binder, 然后dyld_stub_binder负责找到真实的方法，并且将地址bind到桩上，下一次就不用再bind了。
    * 
* objc SetUp：
    * 读取二进制文件的 DATA 段内容，找到与 objc 相关的信息
    * 注册 Objc 类，ObjC Runtime 需要维护一张映射类名与类的全局表。当加载一个 dylib 时，其定义的所有的类都需要被注册到这个全局表中；
    *  读取 protocol 以及 category 的信息，把category的定义插入方法列表 (category registration)，
    * 确保 selector 的唯一性
*  initializers， 按照继承关系加载+load方法，包括（category中的）， 加载c/c++的__attribute__(constructor),或static initialzer方法
    * dyld开始将程序二进制文件初始化
    * 交由ImageLoader读取image，其中包含了我们的类、方法等各种符号
    * 由于runtime向dyld绑定了回调，当image加载到内存后，dyld会通知runtime进行处理
    * runtime接手后调用mapimages做解析和处理，接下来loadimages中调用 callloadmethods方法，遍历所有加载进来的Class，按继承层级依次调用Class的+load方法和其 Category的+load方法
* 整个事件由dyld主导，完成运行环境的初始化后，配合ImageLoader 将二进制文件按格式加载到内存，动态链接依赖库，并由runtime负责加载成objc 定义的结构，所有初始化工作结束后，dyld调用真正的main函数

### pre_main 测量

    测试启动时间，Xcode 提供了一个很赞的方法，只需要在 Edit scheme -> Run -> Arguments 中将环境变量 DYLD_PRINT_STATISTICS 设为 1，就可以看到 main 之前各个阶段的时间消耗。
```
Total pre-main time: 341.32 milliseconds (100.0%)
         dylib loading time: 154.88 milliseconds (45.3%)
        rebase/binding time:  37.20 milliseconds (10.8%)
            ObjC setup time:  52.62 milliseconds (15.4%)
           initializer time:  96.50 milliseconds (28.2%)
           slowest intializers :
               libSystem.dylib :   4.07 milliseconds (1.1%)
    libMainThreadChecker.dylib :  30.75 milliseconds (9.0%)
                  AFNetworking :  19.08 milliseconds (5.5%)
                        LDXLog :  10.06 milliseconds (2.9%)
                        Bigger :   7.05 milliseconds (2.0%)
/// 还有一个方法获取更详细的时间，只需将环境变量 DYLD_PRINT_STATISTICS_DETAILS 设为 1 就可以。
  total time: 1.0 seconds (100.0%)
  total segments mapped: 721, into 93608 pages with 6173 pages pre-fetched
  total images loading time: 817.51 milliseconds (78.3%)
```


### 如何统计+load耗时

两种，一种是通过runtime api，去读取对应镜像下所有类及其元类，并逐个遍历元类的实例方法，如果方法名称为load，则执行hook操作，代表库是AppStartTime；一种是和 runtime一样，直接通过getsectiondata函数，读取编译时期写入mach-o文件DATA段的__objc_nlclslist和 __objc_nlcatlist节，这两节分别用来保存no lazy class列表和no lazy category列表，所谓的no lazy结构，就是定义了+load方法的类或分类，代表库是A4LoadMeasure。
