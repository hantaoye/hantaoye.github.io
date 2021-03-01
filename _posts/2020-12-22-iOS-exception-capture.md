---
title:  "iOS 异常捕获"
date:   2020-12-22 22:11:36 +0530
author: taoye
categories: [iOS, 2020年复现总结]
tags: [iOS]
---

### 异常捕获
**异常类型**
 - Mach 异常：EXC_CRASH
 - UNIX 信号：SIGABRT
 - NSException 异常：应用层，通过 NSUncaughtExceptionHandler 捕获
 
**异常捕获**
 - 利用 Mach API 捕获 Mach 异常
 - 通过 POSIX（可移植性操作系统接口） API 注册 signal(SIGSEGV,signalHandler) 来捕获 UNIX 异常 
**信号**
 - 注册 NSUncaughtExceptionHandler 来捕获应用级异常
 
 ![682cb9e7ad927fa2a940ca0d8ed461a0](/assets/img/exception-capture/WeChat682cb9e7ad927fa2a940ca0d8ed461a0.png)
 
 所有 Mach 异常都在 host 层被ux_exception转换为相应的 Unix 信号，并通过threadsignal将信号投递到出错的线程。iOS 中的 POSIX API 就是通过 Mach 之上的 BSD 层实现的。

 EXC_BAD_ACCESS (SIGSEGV)表示的意思是：Mach 层的EXC_BAD_ACCESS异常，在 host 层被转换成 SIGSEGV 信号投递到出错的线程。既然最终以信号的方式投递到出错的线程，那么就可以通过注册 signalHandler 来捕获信号:

signal(SIGSEGV,signalHandler);

Mach 异常 +Unix 信号方式： Github 上多数开源项目都采用的这种方式，即使在优选捕获 Mach 异常的情况下，也放弃捕获EXC_CRASH异常，而选择捕获与之对应的 SIGABRT 信号
为什么需要两种方式：（内核发送一个EXC_CRASH mach 异常来表示 SIGABRT 终止。在这种情况下，捕获进程中的 Mach 异常会导致进程死锁在不间断的等待中。因此，我们回退 SIGABRT 的 BSD 信号处理程序，并且不注册EXC_CRASH。）

 ### Crash 收集
 
 
 
 ### crash类型
 ![35a010a6dbebd45a38c89c18b968e5ed](/assets/img/exception-capture/6CF338F2-964F-459B-9A12-1B3D5E485FD1.png)
 SIGILL 执行了非法指令，一般是可执行文件出现了错误
 SIGTRAP 断点指令或者其他trap指令产生
 SIGABRT 调用abort产生
 SIGBUS 非法地址。比如错误的内存类型访问、内存地址对齐等
 SIGSEGV 非法地址。访问未分配内存、写入没有写权限的内存等
 SIGFPE 致命的算术运算。比如数值溢出、NaN数值等


 ### 野指针
 访问野指针是不会Crash的，只有野指针指向的地址被写上了有问题的数据才会引发Crash
 
 ![28390a530b6f04130d3d5e677edf4ed5](/assets/img/exception-capture/ff68afc0-2d56-4753-9f7b-c8df0dd14136.png)
 

[野指针放大iOS监控-野指针定位](https://www.jianshu.com/p/4c8a68bd066c)
 **[如何定位Obj-C野指针随机Crash(一)：先提高野指针Crash率](https://cloud.tencent.com/developer/article/1070505)**
   - 对象释放后在内存上填上不可访问的数据，其实这种技术其实一直都有，xcode的Enable Scribble就是这个作用。
   - 要重写对象释放的接口，重写哪个呢？NSObject的dealloc、runtime的 object_dispose，C的free应该都是可以，但是各有优点，我选择的是覆盖面最广的free，free是C的函数，重写了它之后还可以顺带解决一部分C的野指针问题。
   - 填充的不可访问的数据的长度怎么确定？获取内存长度的接口不在标准库中，好在在Mac和iOS中可以用malloc_size就可以。
   - 填什么？和xcode一样，填0x55。

 [如何定位Obj-C野指针随机Crash(二)：让非必现Crash变成必现](https://blog.csdn.net/tencent_bugly/article/details/46374401)
   - 当free被调用的时候我们不真的调用free，而是自己保留着内存，这样系统不知道这片内存已经不需要用了，自然就不会被再次写上别的数据
   - 为了防止系统内存过快耗尽，还需要额外多做几件事：
     1.自己保留的内存大于一定值的时候就释放一部分，防止被系统杀死。
     2.系统内存警告的时候，也要释放一部分内存。
        - 最多存100M， 最多存10*1024*1024个指针，每次释放多少个指针
   - 具体代码看博客

[如何定位Obj-C野指针随机Crash(三)：加点黑科技让Crash自报家门](https://blog.csdn.net/Tencent_Bugly/article/details/46545155)
我们需要自己写一个类，用它的isa来替换已经释放的对象的isa。如果不出我们所料，我们用自己的类覆盖之后，之前调用的sel就换成了调用我们自己的类的某个sel



> 性能优化可以分为：崩溃治理、内存优化、cpu优化、启动优化、卡顿优化（耗时优化）、APM相关（fps、卡顿、cpu监控）
> 主线程占1M,子线程占用512KB