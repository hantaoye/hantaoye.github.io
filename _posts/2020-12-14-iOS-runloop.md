---
title:  "iOS runloop 相关梳理"
date:   2020-12-14 22:11:36 +0530
author: taoye
categories: [iOS, 2020年复现总结]
tags: [iOS, runloop]
---

### runloop介绍
```
function loop() {
    initialize();
    do {
        var message = get_next_message();
        process_message(message);
    } while (message != quit);
}
```
这种模型通常被称作 Event Loop。 Event Loop 在很多系统和框架里都有实现，比如 Node.js 的事件处理，比如 Windows 程序的消息循环，再比如 OSX/iOS 里的 RunLoop。实现这种模型的关键点在于：如何管理事件/消息，如何让线程在没有处理消息时休眠以避免资源占用、在有消息到来时立刻被唤醒。
所以，RunLoop 实际上就是一个对象，这个对象管理了其需要处理的事件和消息，并提供了一个入口函数来执行上面 Event Loop 的逻辑。线程执行了这个函数后，就会一直处于这个函数内部 “接受消息->等待->处理” 的循环中，直到这个循环结束（比如传入 quit 的消息），函数返回。
OSX/iOS 系统中，提供了两个这样的对象：NSRunLoop 和 CFRunLoopRef。
CFRunLoopRef 是在 CoreFoundation 框架内的，它提供了纯 C 函数的 API，所有这些 API 都是线程安全的。
NSRunLoop 是基于 CFRunLoopRef 的封装，提供了面向对象的 API，但是这些 API 不是线程安全的。

在 CoreFoundation 里面关于 RunLoop 有5个类:
CFRunLoopRef
CFRunLoopModeRef
CFRunLoopSourceRef
CFRunLoopTimerRef
CFRunLoopObserverRef
其中 CFRunLoopModeRef 类并没有对外暴露，只是通过 CFRunLoopRef 的接口进行了封装。他们的关系如下:
![28b6c3b13e08212a1f2920f299640c8d](/assets/img/runloop/CA7BF057-C0C1-4570-970F-0B30C6D16C12.png)

一个 RunLoop 包含若干个 Mode，每个 Mode 又包含若干个 Source/Timer/Observer。每次调用 RunLoop 的主函数时，只能指定其中一个 Mode，这个Mode被称作 CurrentMode。如果需要切换 Mode，只能退出 Loop，再重新指定一个 Mode 进入。这样做主要是为了分隔开不同组的 Source/Timer/Observer，让其互不影响。
* CFRunLoopSourceRef 是事件产生的地方。Source有两个版本：Source0 和 Source1。
    - Source0 只包含了一个回调（函数指针），它并不能主动触发事件。使用时，你需要先调用 CFRunLoopSourceSignal(source)，将这个 Source 标记为待处理，然后手动调用 CFRunLoopWakeUp(runloop) 来唤醒 RunLoop，让其处理这个事件。
    - Source1 包含了一个 mach_port 和一个回调（函数指针），被用于通过内核和其他线程相互发送消息。这种 Source 能主动唤醒 RunLoop 的线程
* CFRunLoopTimerRef 是基于时间的触发器，它和 NSTimer 是toll-free bridged 的，可以混用。其包含一个时间长度和一个回调（函数指针）。当其加入到 RunLoop 时，RunLoop会注册对应的时间点，当时间点到时，RunLoop会被唤醒以执行那个回调。
* CFRunLoopObserverRef 是观察者，每个 Observer 都包含了一个回调（函数指针），当 RunLoop 的状态发生变化时，观察者就能通过回调接受到这个变化。可以观测的时间点有以下几个：
```
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry         = (1UL << 0), // 即将进入Loop
    kCFRunLoopBeforeTimers  = (1UL << 1), // 即将处理 Timer
    kCFRunLoopBeforeSources = (1UL << 2), // 即将处理 Source
    kCFRunLoopBeforeWaiting = (1UL << 5), // 即将进入休眠
    kCFRunLoopAfterWaiting  = (1UL << 6), // 刚从休眠中唤醒
    kCFRunLoopExit          = (1UL << 7), // 即将退出Loop
};

struct __CFRunLoopMode {
    CFStringRef _name;            // Mode Name, 例如 @"kCFRunLoopDefaultMode"
    CFMutableSetRef _sources0;    // Set
    CFMutableSetRef _sources1;    // Set
    CFMutableArrayRef _observers; // Array
    CFMutableArrayRef _timers;    // Array
    ...
};
 
struct __CFRunLoop {
    CFMutableSetRef _commonModes;     // Set
    CFMutableSetRef _commonModeItems; // Set<Source/Observer/Timer>
    CFRunLoopModeRef _currentMode;    // Current Runloop Mode
    CFMutableSetRef _modes;           // Set
    ...
};

```

主线程的 RunLoop 里有两个预置的 Mode：kCFRunLoopDefaultMode 和 UITrackingRunLoopMode。这两个 Mode 都已经被标记为”Common”属性
CFRunLoop对外暴露的管理 Mode 接口只有下面2个:

- CFRunLoopAddCommonMode(CFRunLoopRef runloop, CFStringRef modeName);
- CFRunLoopRunInMode(CFStringRef modeName, ...);

Mode 暴露的管理 mode item 的接口有下面几个：
```
CFRunLoopAddSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFStringRef modeName);
CFRunLoopAddObserver(CFRunLoopRef rl, CFRunLoopObserverRef observer, CFStringRef modeName);
CFRunLoopAddTimer(CFRunLoopRef rl, CFRunLoopTimerRef timer, CFStringRef mode);
CFRunLoopRemoveSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFStringRef modeName);
CFRunLoopRemoveObserver(CFRunLoopRef rl, CFRunLoopObserverRef observer, CFStringRef modeName);
CFRunLoopRemoveTimer(CFRunLoopRef rl, CFRunLoopTimerRef timer, CFStringRef mode);
```
你只能通过 mode name 来操作内部的 mode，当你传入一个新的 mode name 但 RunLoop 内部没有对应 mode 时，RunLoop会自动帮你创建对应的 CFRunLoopModeRef。对于一个 RunLoop 来说，其内部的 mode 只能增加不能删除。
苹果公开提供的 Mode 有两个：kCFRunLoopDefaultMode (NSDefaultRunLoopMode) 和 UITrackingRunLoopMode，你可以用这两个 Mode Name 来操作其对应的 Mode。
同时苹果还提供了一个操作 Common 标记的字符串：kCFRunLoopCommonModes (NSRunLoopCommonModes)，你可以用这个字符串来操作 Common Items，或标记一个 Mode 为 “Common”。使用时注意区分这个字符串和其他 mode name。

### Runloop 事件
其实在主线程的任何一个断点，都可以看到调用堆栈信息里面都有CFRunLoop这家伙的影子，一般都是由以下6个回调上来的：

1.
///触发 Source0 (非基于port的) 回调，处理如UIEvent，CFSocket这类事件。需要手动触发。
///触摸事件其实是Source1接收系统事件后在回调__IOHIDEventSystemClientQueueCallback()内触发的Source0
__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__
2.
///如果如果Runloop是被 Source1 (基于port的) 的事件唤醒了，处理这个事件
///处理系统内核的mach_msg事件
__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__
3.  
///被dispatch唤醒，执行放入main_queue的block
__CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__
4.
///被timer唤醒
__CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__
5.
///通知observer当前runloop的状态
__CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__
6.
///非延迟的block事件调用，CFRunLoopPerformBlock，立即执行一个block
__CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__


### RunLoop 的内部逻辑

![7aab9f00eb57aec05e3b18a3d2086374](/assets/img/runloop/3CE7FCCD-C423-4FAA-BE9E-5220BF1AA752.png)
![9fcfa628373f496d0747b334c96f42ff](/assets/img/runloop/835216A3-48E1-48F5-ABF3-C0C1139CB9DD.png)


1. 通知 observer，进入runloop
2. 通知 observer，即将处理 timer 事件
3. 通知 observer，即将处理 source 事件
4. 处理block，runloop 是可以添加 block 的(CFRunLoopPerformBlock(&lt;#CFRunLoopRef rl#&gt;, &lt;#CFTypeRef mode#&gt;, &lt;#^(void)block#&gt;))
5. 处理 source 0
6. 处理 source1事件，如果存在就处理，跳转到第8
7. 通知 observer 即将休眠
8. 通知 observer 结束休眠（timer 事件，source1 事件等）
9. 然后根据前面的执行结果，决定是否再次循环

```
/// 用DefaultMode启动
void CFRunLoopRun(void) {
    CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);
}
 
/// 用指定的Mode启动，允许设置RunLoop超时时间
int CFRunLoopRunInMode(CFStringRef modeName, CFTimeInterval seconds, Boolean stopAfterHandle) {
    return CFRunLoopRunSpecific(CFRunLoopGetCurrent(), modeName, seconds, returnAfterSourceHandled);
}
 
/// RunLoop的实现
int CFRunLoopRunSpecific(runloop, modeName, seconds, stopAfterHandle) {
    
    /// 首先根据modeName找到对应mode
    CFRunLoopModeRef currentMode = __CFRunLoopFindMode(runloop, modeName, false);
    /// 如果mode里没有source/timer/observer, 直接返回。
    if (__CFRunLoopModeIsEmpty(currentMode)) return;
    
    /// 1. 通知 Observers: RunLoop 即将进入 loop。
    __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopEntry);
    
    /// 内部函数，进入loop
    __CFRunLoopRun(runloop, currentMode, seconds, returnAfterSourceHandled) {
        
        Boolean sourceHandledThisLoop = NO;
        int retVal = 0;
        do {
 
            /// 2. 通知 Observers: RunLoop 即将触发 Timer 回调。
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeTimers);
            /// 3. 通知 Observers: RunLoop 即将触发 Source0 (非port) 回调。
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeSources);
            /// 执行被加入的block, 处理非延迟的主线程调用
            /// 一个循环中会调用两次，确保非延迟的NSObject PerformSelector调用和非延迟的dispatch_after调用在当前runloop执行。还有回调block 不知道哪个是对的
            __CFRunLoopDoBlocks(runloop, currentMode);
            
            /// 4. RunLoop 触发 Source0 (非port) 回调。 // 处理UIevent等
            sourceHandledThisLoop = __CFRunLoopDoSources0(runloop, currentMode, stopAfterHandle);
            /// 执行被加入的block
            __CFRunLoopDoBlocks(runloop, currentMode);
 
            /// 5. 如果有 Source1 (基于port) 处于 ready 状态，直接处理这个 Source1 然后跳转去处理消息。
            if (__Source0DidDispatchPortLastTime) {
                Boolean hasMsg = __CFRunLoopServiceMachPort(dispatchPort, &msg)
                if (hasMsg) goto handle_msg;
            }
            
            /// 通知 Observers: RunLoop 的线程即将进入休眠(sleep)。
            if (!sourceHandledThisLoop) {
                __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeWaiting);
            }
            
            /// 7. 调用 mach_msg 等待接受 mach_port 的消息。线程将进入休眠, 直到被下面某一个事件唤醒。
            /// • 一个基于 port 的Source 的事件。
            /// • 一个 Timer 到时间了
            /// • RunLoop 自身的超时时间到了
            /// • 被其他什么调用者手动唤醒
            __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort) {
                mach_msg(msg, MACH_RCV_MSG, port); // thread wait for receive msg
            }
 
            /// 8. 通知 Observers: RunLoop 的线程刚刚被唤醒了。
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopAfterWaiting);
            
            /// 收到消息，处理消息。
            handle_msg:
 
            /// 9.1 如果一个 Timer 到时间了，触发这个Timer的回调。
            if (msg_is_timer) {
                __CFRunLoopDoTimers(runloop, currentMode, mach_absolute_time())
            } 
 
            /// 9.2 如果有dispatch到main_queue的block，执行block。
            else if (msg_is_dispatch) {
                __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
            } 
 
            /// 9.3 如果一个 Source1 (基于port) 发出事件了，处理这个事件
            else {
                CFRunLoopSourceRef source1 = __CFRunLoopModeFindSourceForMachPort(runloop, currentMode, livePort);
                sourceHandledThisLoop = __CFRunLoopDoSource1(runloop, currentMode, source1, msg);
                if (sourceHandledThisLoop) {
                    mach_msg(reply, MACH_SEND_MSG, reply);
                }
            }
            
            /// 执行加入到Loop的block // 再次确保是否有同步的方法需要调用
            __CFRunLoopDoBlocks(runloop, currentMode);
            
 
            if (sourceHandledThisLoop && stopAfterHandle) {
                /// 进入loop时参数说处理完事件就返回。
                retVal = kCFRunLoopRunHandledSource;
            } else if (timeout) {
                /// 超出传入参数标记的超时时间了
                retVal = kCFRunLoopRunTimedOut;
            } else if (__CFRunLoopIsStopped(runloop)) {
                /// 被外部调用者强制停止了
                retVal = kCFRunLoopRunStopped;
            } else if (__CFRunLoopModeIsEmpty(runloop, currentMode)) {
                /// source/timer/observer一个都没有了
                retVal = kCFRunLoopRunFinished;
            }
            
            /// 如果没超时，mode里没空，loop也没被停止，那继续loop。
        } while (retVal == 0);
    }
    
    /// 10. 通知 Observers: RunLoop 即将退出。
    __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);
}

```
分析一下这两者的区别

```
RunLoop执行顺序的伪代码

SetupThisRunLoopRunTimeoutTimer(); // by GCD timer

//通知即将进入runloop__CFRUNLLOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(KCFRunLoopEntry);

do {

     __CFRunLoopDoObservers(kCFRunLoopBeforeTimers);

     __CFRunLoopDoObservers(kCFRunLoopBeforeSources);

     __CFRunLoopDoBlocks();  //一个循环中会调用两次，确保非延迟的NSObject PerformSelector调用和非延迟的dispatch_after调用在当前runloop执行。还有回调block

     __CFRunLoopDoSource0(); //例如UIKit处理的UIEvent事件

     CheckIfExistMessagesInMainDispatchQueue(); //GCD dispatch main queue

     __CFRunLoopDoObservers(kCFRunLoopBeforeWaiting); //即将进入休眠，会重绘一次界面

     var wakeUpPort = SleepAndWaitForWakingUpPorts();

     // mach_msg_trap，陷入内核等待匹配的内核mach_msg事件

     // Zzz...

     // Received mach_msg, wake up

     __CFRunLoopDoObservers(kCFRunLoopAfterWaiting);

     // Handle msgs

     if (wakeUpPort == timerPort) {

          __CFRunLoopDoTimers();

     } else if (wakeUpPort == mainDispatchQueuePort) {

          //GCD当调用dispatch_async(dispatch_get_main_queue(),block)时，libDispatch会向主线程的runloop发送mach_msg消息唤醒runloop，并在这里执行。这里仅限于执行dispatch到主线程的任务，dispatch到其他线程的仍然是libDispatch来处理。

          __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__()

     } else {

          __CFRunLoopDoSource1();  //CADisplayLink是source1的mach_msg触发？

     }

     __CFRunLoopDoBlocks();

} while (!stop && !timeout);

//通知observers，即将退出runloop

__CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBERVER_CALLBACK_FUNCTION__(CFRunLoopExit);

结合上面的Runloop事件执行顺序，思考下面代码逻辑中为什么可以标识tableview是否reload完成

dispatch_async(dispatch_get_main_queue(), ^{

    _isReloadDone = NO;

    [tableView reload]; //会自动设置tableView layoutIfNeeded为YES，意味着将会在runloop结束时重绘table

    dispatch_async(dispatch_get_main_queue(),^{

        _isReloadDone = YES;

    });

});
// 这里在GCD dispatch main queue中插入了两个任务，一次RunLoop有两个机会执行GCD dispatch main queue中的任务，分别在休眠前和被唤醒后。
```

当你调用 CFRunLoopRun() 时，线程就会一直停留在这个循环里；直到超时或被手动停止，该函数才会返回。
RunLoop 的核心就是一个 mach_msg() (见上面代码的第7步)，RunLoop 调用这个函数去接收消息，如果没有别人发送 port 消息过来，内核会将线程置于等待状态。例如你在模拟器里跑起一个 iOS 的 App，然后在 App 静止时点击暂停，你会看到主线程调用栈是停留在 mach_msg_trap() 这个地方。

### 卡顿检查

从上可以看出RunLoop处理事件的时间主要出在两个阶段：
* kCFRunLoopBeforeSources和kCFRunLoopBeforeWaiting之间
* kCFRunLoopAfterWaiting之后
所以监测下面两个状态
if (self->_activity == kCFRunLoopBeforeSources || self->_activity == kCFRunLoopAfterWaiting) {

###  事件响应

系统注册了一个 Source1 用来接收系统事件，其回调函数为 __IOHIDEventSystemClientQueueCallback()。当一个硬件事件(触摸/锁屏/摇晃等)发生后，首先由 IOKit.framework 生成一个 IOHIDEvent 事件并由 SpringBoard 接收。
(SpringBoard 只接收按键(锁屏/静音等)、触摸、加速，传感器等几种事件)
随后用 mach port 转发给需要的App进程。随后系统注册的那个 Source1 就会触发回调，并调用_UIApplicationHandleEventQueue()进行应用内部的分发。 _UIApplicationHandleEventQueue()会把 IOHIDEvent 事件处理并包装成 UIEvent 进行处理或分发，其中包括识别 UIGesture/处理屏幕旋转/发送给 UIWindow 等。通常事件比如 UIButton 点击、touchesBegin/Move/End/Cancel 事件都是在这个回调中完成的。

###  界面更新

iOS 的显示系统是由 VSync 信号驱动的，VSync 信号由硬件时钟生成，每秒钟发出 60 次（这个值取决设备硬件，比如 iPhone 真机上通常是 59.97）。iOS 图形服务接收到 VSync 信号后，会通过 IPC 通知到 App 内。App 的 Runloop 在启动后会注册对应的 CFRunLoopSource 通过 mach_port 接收传过来的时钟信号通知，随后 Source 的回调会驱动整个 App 的动画与显示。

Core Animation 在 RunLoop 中注册了一个 Observer，监听了 BeforeWaiting 和 Exit 事件。当一个触摸事件到来时，RunLoop 被唤醒，App 中的代码会执行一些操作，比如创建和调整视图层级、设置 UIView 的 frame、修改 CALayer 的透明度、为视图添加一个动画；这些操作最终都会被 CALayer 标记，并通过 CATransaction 提交到一个中间状态去。当上面所有操作结束后，RunLoop 即将进入休眠（或者退出）时，关注该事件的 Observer 都会得到通知。这时 Core Animation 注册的那个 Observer 就会在回调中，把所有的中间状态合并提交到 GPU 去显示；如果此处有动画，通过 DisplayLink 稳定的刷新机制会不断的唤醒runloop，使得不断的有机会触发observer回调，从而根据时间来不断更新这个动画的属性值并绘制出来。
为了不阻塞主线程，Core Animation 的核心是 OpenGL ES 的一个抽象物，所以大部分的渲染是直接提交给GPU来处理

在操作 UI 时，比如改变了 Frame、更新了 UIView/CALayer 的层次时，或者手动调用了 UIView/CALayer 的 setNeedsLayout/setNeedsDisplay方法后，这个 UIView/CALayer 就被标记为待处理，并被提交到一个全局的容器去。

苹果注册了一个 Observer 监听 BeforeWaiting(即将进入休眠) 和 Exit (即将退出Loop) 事件，回调去执行一个很长的函数：_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv()。这个函数里会遍历所有待处理的 UIView/CAlayer 以执行实际的绘制和调整，并更新 UI 界面。
这个函数内部的调用栈大概是这样的：
```
_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv()
    QuartzCore:CA::Transaction::observer_callback:
        CA::Transaction::commit();
            CA::Context::commit_transaction();
                CA::Layer::layout_and_display_if_needed();
                    CA::Layer::layout_if_needed();
                        [CALayer layoutSublayers];
                            [UIView layoutSubviews];
                    CA::Layer::display_if_needed();
                        [CALayer display];
                            [UIView drawRect];

```

### RunLoop与GCD关系

1. RunLoop 的超时时间就是使用 GCD 中的&nbsp;
2. dispatch_source_t来实现的内部注册了GCD的系统事件，调用了dispatch_async(dispatch_get_main_queue(), &lt;#^(void)block#&gt;)时，libDispatch会向主线程RunLoop发送消息唤醒RunLoop，RunLoop从消息中获取block，并且在__CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__回调里执行这个block。dispatch_after同理。如图：
![5895d1a1bdc995ef5de15b2b82e68f3b](/assets/img/runloop/4F34EEC6-30CB-4703-AA00-0F243EEA6018.png)

### 线程保活的三种方式
1. [runLoop addPort:[NSMachPort port] forMode:NSRunLoopCommonModes];
2. 子线程创建timer，repeats=yes，其实就是让runloop一直处于有接受timer source的状态条件锁或信号量，子线程里写一个循环，/** 让子线程任务进入闲等状态 
3. 
```
- (void)conditionThread { 
  do {
    [self.conditon lock];
    if (self.myTask) {
        self.myTask();
        self.myTask = nil;
    }
    [self.conditon wait];
    [self.conditon unlock];
  } while (!self.isExitThread);
}
 - (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    self.myTask = ^{
        NSLog(@"条件锁常驻线程 %@", NSThread.currentThread);
    };
 }
```
 ### runloop的坑 （怎么退出runloop）
 NSRunloop: &nbsp;run run:until: 
 这两个个方法, 即使 remove 所有 sources, timers , 也不会立刻exit。调用 stop没用;
 run(mode:before:); 调用stop会exit，但按照官方文档说不是实时的
 CFRunLoop
 CFRunLoopRun();和CFRunLoopRunInMode(_ : _ : _ :);run 的RunLoop;, 可以使用CFRunLoopStop(_:)来立即停止

Layer也和View一样存在着一个层级树状结构,称之为图层树(Layer Tree),直接创建的或者通过UIView获得的(view.layer)用于显示的图层树,称之为模型树(Model Tree),模型树的背后还存在两份图层树的拷贝,一个是呈现树(Presentation Tree),一个是渲染树(Render Tree). 呈现树可以通过普通layer(其实就是模型树)的layer.presentationLayer获得,而模型树则可以通过modelLayer属性获得(详情文档).模型树的属性在其被修改的时候就变成了新的值,这个是可以用代码直接操控的部分;呈现树的属性值和动画运行过程中界面上看到的是一致的.而渲染树是私有的,你无法访问到,渲染树是对呈现树的数据进行渲染,为了不阻塞主线程,渲染的过程是在单独的进程或线程中进行的,所以你会发现Animation的动画并不会阻塞主线程.
layoutIfNeeded就是直接修改了模型树的数据


 ### runloop 线程和 AutoRelease的关系
 
 如果当前线程没有AutorelesepoolPage的话，代码执行顺序为autorelease -> autoreleaseFast -> autoreleaseNoPage。
 在autoreleaseNoPage方法中，会创建一个hotPage，然后调用page->add(obj)。也就是说即使这个线程没有AutorelesepoolPage，使用了autorelease对象时也会new一个AutoreleasepoolPage出来管理autorelese对象，不用担心内存泄漏。
 
 1.子线程在使用autorelease对象时，如果没有autoreleasepool会在autoreleaseNoPage中懒加载一个出来。
 2.在runloop的run:beforeDate，以及一些source的callback中，有autoreleasepool的push和pop操作，总结就是系统在很多地方都差不多autorelease的管理操作。
 3.就算插入没有pop也没关系，在线程exit的时候会释放资源，执行AutoreleasePoolPage::tls_dealloc，在这里面会做pop操作
 
 ![59fc875d90f5db432df763a6dacae749](/assets/img/runloop/F476B1B1-2CC7-43DC-B5D5-295423B5CF59.png)
 
 ```
 void kill()从双向链表中移除当前AutoreleasePoolPage节点后的所有节点，包括节点本身。注意kill()不包含释放对象的操作，只是简单移除节点。
 void kill() 
{
    AutoreleasePoolPage *page = this;
    while (page->child) page = page->child;

    AutoreleasePoolPage *deathptr;
    do {
        deathptr = page;
        page = page->parent;
        if (page) {
            page->child = nil;
        }
        delete deathptr;
    } while (deathptr != this);
}
 ```
 
 
 前文提及 hotPage 和 codePage 的实现，线程中私有空间中保存了 hotPage 的地址，因此在不同的线程上调用AutoreleasePoolPage类的hotPage()静态方法时，返回的是不同的AutoreleasePoolPage实例。AutoreleasePoolPage双向链表中的所有节点的堆栈空间，实际是统一的整体，它是一条线程上创建的所有 autorelease pool 的堆栈。
线程与AutoreleasePoolPage的关系如下图所示。假设App使用了三条线程主线程Thread_Main、后台线程Thread_A及Thread_B，其中红色箭头表示线程与AutoreleasePoolPage之间的关联，线程通过私有空间中的 Key-Value 映射可以获取到该线程的 hotPage，AutoreleasePoolPage 通过thread指针可获取其关联线程；蓝色箭头表示AutoreleasePoolPage之间使用双向链表通过parent、child指针关联；

![a62063e01ff3b86005d2fd2258d23fda](/assets/img/runloop/e3459ce4-7e33-4d88-af68-40b5b17edea0.png)



### runloop和CADisplayLink
CADisplayLink是一个以屏幕刷新频率同步的计时器。以下是创建方法：

创建方法
```
self.displayLink = [CADisplayLink displayLinkWithTarget:self selector:@selector(handleDisplayLink:)];    
[self.displayLink addToRunLoop:[NSRunLoop currentRunLoop] forMode:NSDefaultRunLoopMode];
停止方法
[self.displayLink invalidate];  
self.displayLink = nil;
```
CADisplayLink的计算时间并不依靠RunLoop，当一个屏幕刷新完成时候则会通知RunLoop给对应的target执行action。但是CADisplayLink依然会有精度的问题，当两次界面刷新之间执行了一次长任务的时候，那就会有一帧被跳过去，也就是所谓的掉帧，那相应的此次也不会调用target的action。

CADisplayLink是通过mach port 唤起的source1，然后执行到display_timer_callback里，再执行对应的displayLink进行响应target的selecter调度
![8311b0fb74e48e2b38c704605ade748b](/assets/img/runloop/0AD45D36-CCB2-46FF-AF34-47AD477E6FD5.png)

timer的调度是由timermode唤醒的，两者不一致，验证了上面所述内容
![b95879339247a41347952c632531666f](/assets/img/runloop/3b1ac4a5-4069-40fb-b2a9-ab9878e1273d.png)
