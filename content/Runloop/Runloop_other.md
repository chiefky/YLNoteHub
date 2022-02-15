# 1. Runloop 与 AutoReleasePool

前言

理解AutoReleasePool与AutoReleasePoolPage：

> - AutoreleasePool并没有单独的结构，而是由若干个AutoreleasePoolPage以双向链表的形式组合而成（分别对应结构中的parent指针和child指针）
>
> - AutoreleasePool 的释放（不是销毁，虚拟名词不存在销毁一说）是指：
>
>   1. 根据传入的哨兵对象地址找到哨兵对象所处的page
>   2. 从最新加入的对象一直向前清理，可以向前跨越若干个page，直到哨兵所在的page，将晚于哨兵对象插入的所有autorelease对象都发送一次- release消息，并移动next指针到正确位置
>
> - 自动释放池的压栈和出栈主要是通过结构体的构造函数和析构函数调用底层的objc_autoreleasePoolPush和objc_autoreleasePoolPop，实际上是调用AutoreleasePoolPage的push和pop两个方法
>
>   - push 操作内部调用了 AutoreleasePoolPage 的 autoreleaseFast() 方法，该方法会根据 hot page(通过 TLS 获取 ) 的当前状态，进行对应的处理，并向最终定位的 page 中插入一个POOL_BOUNDARY，并返回插入POOL_BOUNDARY的内存地址。autoreleaseFast()方法的内部处理逻辑有以下三种情况:
>
>   - - 当hot page存在，且不满时，调用add方法将对象添加至page的next指针处，并next递增
>     - 当 hot page存在，且已满时，调用autoreleaseFullPage初始化一个新的page，然后调用add方法将对象添加至page栈中
>     - 当 hot page不存在时，调用autoreleaseNoPage创建一个hotPage，然后调用add方法将对象添加至page栈中
>
>   - 当执行pop操作时，会传入一个值，这个值就是push操作的返回值，即POOL_BOUNDARY的内存地址token。所以pop内部的实现就是根据token找到哨兵对象所处的page中，然后使用 objc_release 释放 token之前的对象，并把next 指针到正确位置

## 1.1 主线程的Autoreleasepool与子线程的Autoreleasepool

#### 主线程的Autoreleasepool：

> App启动后，苹果在主线程 RunLoop 里注册了两个 Observer，其回调都是 _wrapRunLoopWithAutoreleasePoolHandler()。
>
> - 第一个 Observer 监视的事件是 Entry(即将进入Loop)，其回调内会调用 _objc_autoreleasePoolPush() 创建自动释放池。其 order 是-2147483647，优先级最高，保证创建释放池发生在其他所有回调之前。
> - 第二个 Observer 监视了两个事件： BeforeWaiting(准备进入休眠) 时调用_objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush() 释放旧的池并创建新池；Exit(即将退出Loop) 时调用 _objc_autoreleasePoolPop() 来释放自动释放池。这个 Observer 的 order 是 2147483647，优先级最低，保证其释放池子发生在其他所有回调之后。

#### 子线程的Autoreleasepool:

> 子线程会在创建的时候，同时创建一个autoreleasepool（通过调用堆栈可知，无法证明通过什么创建） ，然后在线程kill 掉的时候释放掉；

综上：主线程的runloop系统添加了观察者，会在指定的时机释放池子中的对象，合适的时机创建新的池子插入boundary对象；而子线程只会在首次开启runloop时创建一个池子，并没有加入观察者观察runloop的各个时机，所以子线程中创建的Autorelease对象只能在子线程销毁的时候释放池子中的所有对象。因此这里有一个优化点：当子线程产生大量autorelease对象时通过手动添加Autoreleasepool可以及时释放部分内存开销，从而能够避免出现内存峰值。

例子：

在obj = test处设置断点使用`watchpoint set variable obj`命令观察obj，可以看到obj在释放时的方法调用栈是这样的。

<img src="/Users/tangh/yuki/博客/文章仓库/YLNoteHub/content/images/runloop_usage_autoreleasepool_0.png" alt="runloop_usage_autoreleasepool_0" style="zoom:60%;" />

<img src="/Users/tangh/yuki/博客/文章仓库/YLNoteHub/content/images/runloop_usage_autoreleasepool_1.png" alt="runloop_usage_autoreleasepool_1" style="zoom:60%;" />

## 1.2 main方法为什么会包一层 autoreleasePool？

为了释放 "主线程 runloop 创建 autoreleasepool 之前的对象", 也就是 NSStringFromClass([AppDelegate class]) 内部创建的autorelease对象。

# 2. Runloop 与 事件响应 和 手势识别

苹果注册了一个 Source1 (基于 mach port 的) 用来接收系统事件，其回调函数为 __IOHIDEventSystemClientQueueCallback()。

1. 当一个硬件事件(触摸/锁屏/摇晃等)发生后，首先由 IOKit.framework 生成一个 IOHIDEvent 事件并由 SpringBoard 接收。这个过程的详细情况可以参考[这里](http://iphonedevwiki.net/index.php/IOHIDFamily)。SpringBoard 只接收按键(锁屏/静音等)，触摸，加速，接近传感器等几种 Event，随后用 mach port 转发给需要的App进程。随后苹果注册的那个 Source1 就会接收 IOHIDEvent，并在回调 __IOHIDEventSystemClientQueueCallback() 内触发 Source0 ， Source0 再触发 _UIApplicationHandleEventQueue() 进行应用内部的分发。_
2. UIApplicationHandleEventQueue() 会把 IOHIDEvent 处理并包装成 UIEvent 进行处理或分发，其中包括识别 UIGesture/处理屏幕旋转/发送给 UIWindow 等。通常事件比如 UIButton 点击、touchesBegin/Move/End/Cancel 事件都是在这个回调中完成的。
3.  当 _UIApplicationHandleEventQueue() 识别了一个手势时，其首先会调用 Cancel 将当前的 touchesBegin/Move/End 系列回调打断。随后系统将对应的 UIGestureRecognizer 标记为待处理。
4. 苹果注册了一个 Observer 监测 BeforeWaiting (Loop即将进入休眠) 事件，这个Observer 的回调函数是 _UIGestureRecognizerUpdateObserver()，其内部会获取所有刚被标记为待处理的 GestureRecognizer，并执行GestureRecognizer的回调。
5. 当有 UIGestureRecognizer 的变化(创建/销毁/状态改变)时，这个回调都会进行相应处理。

总结：

   用户交互事件首先在 IOHID 层生成 HIDEvent，然后向事件处理线程的 Source1 的 mach port 发送 HIDEvent 消息，Source1 的回调函数将事件转化为 UIEvent 并筛选需要处理的事件推入待处理事件队列，向主线程的事件处理 Source0 发送信号，并唤醒主线程，主线程检查到事件处理 Source0 有待处理信号后，触发 Source0 的回调函数，从待处理事件队列中提取 UIEvent，最后进入 hit-test 等 UIEvent 事件响应流程。

# **3. Runloop** 与 界面更新

1. 当操作 UI 时，比如改变了 Frame、更新了 UIView/CALayer 的层次时，或者手动调用了 UIView/CALayer 的 setNeedsLayout/setNeedsDisplay方法后，这个 UIView/CALayer 就被标记为待处理，并被提交到一个全局的容器去。
2. 苹果注册了一个 Observer 监听 BeforeWaiting(即将进入休眠) 和 Exit (即将退出Loop) 事件，回调去执行一个很长的函数：
    _ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv()。这个函数里会遍历所有待处理的 UIView/CAlayer 以执行实际的绘制和调整，并更新 UI 界面。这个函数内部的调用栈大概是这样的：

>_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv()
>
>  QuartzCore:CA::Transaction::observer_callback:
>
>​    CA::Transaction::commit();
>
>​      CA::Context::commit_transaction();
>
>​        CA::Layer::layout_and_display_if_needed();
>
>​          CA::Layer::layout_if_needed();
>
>​            [CALayer layoutSublayers];
>
>​              [UIView layoutSubviews];
>
>​          CA::Layer::display_if_needed();
>
>​            [CALayer display];
>
>​              [UIView drawRect];



# **4. Runloop** 与 定时器

参考：

1. [**iOS**刨根问底**-**深入理解**RunLoop**](https://www.cnblogs.com/kenshincui/p/6823841.html)（**RunLoop**应用 **— NSTimer** 部分的代码示例可以看下）
2. [从 RunLoop 源码探索 NSTimer 的实现原理（iOS）](https://toutiao.io/posts/4330zh/preview)（内部包含timer相关源码的详细解读）

<FONT color='red'>重点结论：当前时刻未超过多个 timerInterval 时，timer 触发只会延迟执行，不会丢失。若超过多个 timerInterval 时，只会执行 最早应该触发的那次 timer，它之后的 与 当前时刻之间的其他 触发时机 都会舍弃掉，</font>

## 4.1. 添加**timer**的两种方式：

#### 第一种写法

- NSTimer *timer = [NSTimer timerWithTimeInterval:1.0 target:self selector:@selector(timerUpdate) userInfo:nil repeats:YES];
- [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode];
- [timer fire];

 注意：

- target 参数：timer 将会强引用 target 对象，直到调用 [timer invalidated] 方法
-  [timer invalidated] 方法：从 Runloop 中移除 timer 的唯一方式，移除的同时会去掉 Runloop 对 timer 的强引用，同时还会移除 timer 对 target 的强引用

#### 第二种写法

- [NSTimer scheduledTimerWithTimeInterval:1.0 target:self selector:@selector(timerUpdate) userInfo:nil repeats:YES];

## 4.2 添加 **Timer** 部分逻辑实现解析：

- __CFRepositionTimerInMode：先调用__CFRunLoopInsertionIndexInTimerArray 函数，这个函数就是根据timer的_fireTSR时间字段，利用二分查找的算法，将timer插入到已按照时间排列好的timerArray（rlm_timers）中，这个rlm_timers的array是按照fireTSR的升序排列的。然后再调用__CFArmNextTimerInMode函数.

- __CFArmNextTimerInMode：根据mode中的最前面的那个timer的触发时间，将其通过dispatch_source_set_runloop_timer或者mk_timer的方式注册。具体注册机制是根据RunLoopMode中Timer的时间点和 tolerance，计算出timer触发的SoftDeadline（理论触发的时间点）和HardDeadline（最晚的时间点）：

- - 若二者相同（没有tolerance的情况下），则调用底层xnu的mk_timer注册一个mach-port事件（[具体代码实现](https://opensource.apple.com/source/xnu/xnu-3789.51.2/osfmk/kern/mk_timer.c)）；

- - 若不同（有tolerance ），则调用_dispatch_source_set_runloop_timer_4CF 函数，通过查阅libdispatch的源码可知这个函数就是dispatch_source_set_timer，因此对于有tolerance 的 NSTimer，其最终注册成了一个GCD Timer，只不过最终定时器fire的时候，会再通过RunLoop那一层，调用RunLoopTimer中保存的回调。

## 4.3 **timer**触发 部分逻辑实现解析：

- RunLoop被唤醒后，调用了__CFRunLoopDoTimers函数，这个函数取出所有 _fireTSR(理论触发的时间)小于当前系统时刻的RunLoopTimer，对其分别调用__CFRunLoopDoTimer函数。__CFRunLoopDoTimer函数主要干了两个事情：
  1. 对 timer中存的 callout 进行调用：__CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__(rlt->_callout, rlt, context_info);
  2. 根据 timer中的间隔 interval信息，和当前这次fire的理论触发时刻_fireTSR，计算得到下一个应该触发的时刻_fireTSR，下一个应该触发的时刻_fireTSR必须晚于系统当前时刻，将_fireTSR设置到timer结构中，然后调用__CFRepositionTimerInMode函数，重新排列这个mode中的所有timer触发时刻。

1. 

- - - 延伸结论：

    - - 对于重复的NSTimer，其多次触发的时刻不是一开始算好的，而是timer触发后计算的。但是计算时参考的是上次应当触发的时间_fireTSR，因此计算出的下次触发的时刻不会有误差。这保证了timer不会出现误差叠加。

- - - - 只要阻塞结束后__CFRunLoopDoTimers依然被调用，就不会影响 timer的回调，这一段逻辑不会去校验timer的回调点是否超出了tolerance，因此也不会阻塞掉 RunLoopTimer的触发，所以：

      - - 对于有tolerance的timer的情况，只要仍然能收到GCD timer的mach-port消息，这次timer的回调就会触发，只不过回调触发的时间变晚了不少。
        - 对于没有tolerance的timer，同样，只要能收到mk_timer发出的mach-port时间，就仍然会触发这次timer的回调。

- - - - 如果RunLoop忙的时间过长，以至于收到mach-port消息时，已经过了下次的理论触发点，则系统在__CFRunLoopDoTimer逻辑中计算_fireTSR的时候，会找到晚于当前时刻的那个理应触发点，作为_fireTSR。就是下面这一小段代码：while (nextFireTSR <= currentTSR) {nextFireTSR += intervalTSR;}。因此，如果RunLoop的忙的时间很长，长度达到了好多个timeInteval，则忙的这段时间内的timer回调只会被触发一次。

## 4.4. 总结：

- NSTimer 其实就是 CFRunLoopTimerRef，他们之间是 toll-free bridged 的。

- 对于重复的NSTimer，其多次触发的时刻不是一开始算好的，而是timer触发后计算的。但是计算时参考的是上次理论触发的时间 _fireTSR（不考虑阻塞造成的延迟理论上应该触发的时刻），因此计算出的下次触发的时刻不会有误差。

- Timer 有个属性叫做 Tolerance (宽容度)，标示了当时间点到后，容许有多少最大误差。设置了tolerance的NSTimer，对于iOS和MacOS系统，实质上会采用GCD timer 的形式注册到内核中，GCD timer 触发后，再由RunLoop处理其回调逻辑。对于没有设置tolerance的timer，则是用mk_timer的形式注册。

- Runloop 阻塞未超过多个timer 间隔周期时，不会取消timer 触发。RunLoop层在timer触发后进行回调的时候，不会对tolerance进行验证。也就是说，因为RunLoop忙导致的timer触发时刻超出了tolerance的情况下，timer并不会取消，只会延迟执行。

- 对于RunLoop忙时很长（或者timeInteval很短）的情况，会导致本该在这段时间内触发的几次回调中，只触发一次。也就是说，这种情况下还是会损失回调的次数。（具体是由于当触发timer的时候，会计算下一次触发的时刻，具体实现：while (nextFireTSR <= currentTSR) {nextFireTSR += intervalTSR;}，所以从 当次触发时刻 与 当前时刻 之间的触发时机就被舍弃掉了， 如图：）

<img src="/Users/tangh/yuki/博客/文章仓库/YLNoteHub/content/Runloop/image/Runloop_timer_0.png" alt="NSURLConnection工作原理" style="zoom:90%;" />

- 对于RunLoop比较忙的情况，timer的回调时刻有可能不准，且不会受到tolerance的任何限制。tolerance的作用不是决定timer是否触发的标准，而是一个传递给系统的数值，帮助系统合理的规划GCD Timer的mach-port触发时机。设置了tolerance，一定会损失一定的时间精确度，但是可以显著的降低耗电。

- CADisplayLink 是一个执行频率（fps）和屏幕刷新相同（可以修改preferredFramesPerSecond改变刷新频率）的定时器，它也需要加入到RunLoop才能执行。与NSTimer类似，CADisplayLink同样是基于CFRunloopTimerRef实现，底层使用mk_timer（可以比较加入到RunLoop前后RunLoop中timer的变化）。和NSTimer相比它精度更高（尽管NSTimer也可以修改精度），不过和NStimer类似的是如果遇到大任务它仍然存在丢帧现象。通常。如果在两次屏幕刷新之间执行了一个长任务，那其中就会有一帧被跳过去（和 NSTimer 相似），造成界面卡顿的感觉。

- 非主线程的RunLoop并不会自动运行（同时注意默认情况下非主线程的RunLoop并不会自动创建，直到第一次使用），RunLoop运行必须要在加入NSTimer或Source0、Sourc1、Observer输入后运行否则会直接退出。例如上面代码如果**run**放到NSTimer创建之前则既不会执行定时任务也不会执行循环运算。

- 上面的代码也充分说明了RunLoop是一个循环事实，run方法之后的代码不会立即执行，直到RunLoop退出。

注意：

- 子线程添加的 **NSTimer** 只能在对应子线程中进行停止，若在不同线程调用 **invalidate** 方法，只会停止**timer**回调，但不会真正释放**timer**，从而也无法真正停止对应线程的**runloop**，导致在该**runloop**结束前，对应 **timer** 的内存泄露
- 用 **scheduledTimerWithTimeInterval** 方法创建的 **timer** 不需要再执行 **[****NSRunloop** **addTimer****:** **forMode****: ]** 方法，也能添加到 **runloop** 中
- **scheduledTimerWithTimeInterval** 方法 在 **[****NSRunLoop** **currentRunLoop****]** 之前 或 之后 执行都可以
- **NSTimer** 在非 **repeat**的情况下，会在触发后，自动取消对应 **runloop**的注册，使**runloop**缺少事件源退出。
- **[****NSRunloop run****]** 或 **[****NSRunloop** **runUntilDate****:]** 方法在没有 **input** **source** 或 **timer** 时一样会退出**(**例如：**performSelector****:****afterDelay****:** 方式**)**，但是手动移除 **inputSource** 或 **timer** 不一定能保证它们一定会退出**(**例如：[**timer** **invalidate****]** 方式**)**

# 5. Runloop与performSelector

参考：

1. [**iOS**源码解析**: performSelector**是如何实现的？](https://juejin.cn/post/6844904115185664008#heading-16)
2. [PerformSelector原理](https://blog.chenyalun.com/2018/09/30/PerformSelector原理/#内存泄露)

## 5.1接口

#### 普通方法

- **- (id)performSelector:(SEL)aSelector;**

- **- (id)performSelector:(SEL)aSelector withObject:(id)object;**

- **- (id)performSelector:(SEL)aSelector withObject:(id)object1 withObject:(id)object2;**

- - 内部实现是调用了 objc_msgSend 方法

#### 延迟方法

- **- (void)performSelector:(SEL)aSelector withObject:(nullable id)anArgument afterDelay:(NSTimeInterval)delay;**

- - 当调用 NSObject 的 performSelecter:afterDelay: 后，实际上其内部会创建一个 Timer 并添加到当前线程的 RunLoop 中。所以如果当前线程没有 RunLoop，则这个方法会失效。
  - 该方法会持有调用 **perform** 的对象，直到**selector** 调用结束 或者 直接停止指定线程的 **runloop**

#### 取消延迟

- **+ (void)cancelPreviousPerformRequestsWithTarget:(id)aTarget selector:(SEL)aSelector object:(nullable id)anArgument;**

- **+ (void)cancelPreviousPerformRequestsWithTarget:(id)aTarget;**

- - 调用此方法取消一个延迟任务时，需要与 **performSelector** 调用所在相同的线程执行

#### 线程+模式方法

- **- (void)performSelectorOnMainThread:(SEL)aSelector withObject:(nullable id)arg waitUntilDone:(BOOL)wait modes:(nullable NSArray<NSString \*> \*)array;**

- **- (void)performSelectorOnMainThread(SEL)aSelector withObject:(nullable id)arg waitUntilDone:(BOOL)wait;**

- **- (void)performSelector:(SEL)aSelector onThread:(NSThread \*)thr withObject:(nullable id)arg waitUntilDone:(BOOL)wait modes:(nullable NSArray<NSString \*> \*)array;**

- **- (void)performSelector:(SEL)aSelector onThread:(NSThread \*)thr withObject:(nullable id)arg waitUntilDone:(BOOL)wait；**

- **\- (void)performSelectorInBackground:(SEL)aSelector withObject:(nullable id)arg API_AVAILABLE(macos(10.5), ios(2.0), watchos(2.0), tvos(9.0));**

- 调用该方法分为几种情况：

- - 1. <font color='red'> 当指定的线程 与当前线程相同，且（阻塞当前线程 或 runloop 不存在）时，内部实现与 objc_msgSend 相同</font>

  - 2. <font color='red'>当指定的线程 与当前线程相同，且阻塞当前线程，且 runloop 存在时，内部实现通过 source0</font>
    3. <font color='red'>当指定的线程 非当前线程，且不需要阻塞的话，内部实现通过 source0</font>
    4. <font color='red'>当指定的线程 非当前线程，且需要阻塞的话，会通过 source0 同时 生成一个NSConditionLock 锁阻塞住当前线程</font>

## 5.2 问题Q&A

### printInfo方法会执行吗

#### GCD调用

```objective-c
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    [self performSelector:@selector(printInfo) withObject:nil afterDelay:1];
});
```

很明显是不会的。因为`performSelector`具体实现中并没有主动触发线程对应Runloop运行。子线程对应的Runloop没有run。怎么让它执行？启动Runloop:

```objective-c
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    [self performSelector:@selector(printInfo) withObject:nil afterDelay:1];
    [[NSRunLoop currentRunLoop] run];
});
```

#### NSThread调用

```
NSThread *thread = [[NSThread alloc] initWithBlock:^{
   [self performSelector:@selector(printInfo) withObject:nil afterDelay:1];
}];
[thread start];
```

同理:

```
NSThread *thread = [[NSThread alloc] initWithBlock:^{
   [self performSelector:@selector(printInfo) withObject:nil afterDelay:1];
   [[NSRunLoop currentRunLoop] run];
}];
[thread start];
```

这就有一个问题了。在GCD中`[[NSRunLoop currentRunLoop] run];`放在`performSelector`前面或者后面貌似都是可以的，但是在NSThread中`[[NSRunLoop currentRunLoop] run];`只能放在`performSelector`的后面。

在NSThread方法中:

> 因为run方法只是尝试想要开启当前线程中的runloop，但是如果该线程中并没有任何事件(source、timer、observer)的话，并不会成功的开启。

~~为什么GCD中即使`[[NSRunLoop currentRunLoop] run];`放在前面`printInfo`方法还是调用了呢? 代码实际测试，延迟效果没有了，并且有时方法执行，有时方法没有执行。~~[实际验证，一次都没执行，跟NSThread效果一样]

综上，**在子线程中使用performSelector的延迟方法，需要加上`[[NSRunLoop currentRunLoop] run];`使得Runloop能够运行，并且该方法要放在`performSelector`的后面来保证Runloop成功运行。**

#### NSThread无效

```objective-c
NSThread *thread = [[NSThread alloc] initWithBlock:^{}];
[thread start];
[self performSelector:@selector(printInfo) onThread:thread withObject:nil waitUntilDone:NO];
```

上面这段代码为什么没有执行printMainInfo方法?

子线程执行完操作之后就会立即释放，即使我们使用强引用引用子线程使子线程不被释放，也不能给子线程再次添加操作，或者再次开启。这里可以使用Runloop。子线程获取其对应的Runloop对象并使之运行。一般使用常驻子线程。

正确实践：

```
NSThread *thread = [[NSThread alloc] initWithBlock:^{
   // 执行一次而已
   NSRunLoop *currentRunLoop = [NSRunLoop currentRunLoop];
   [currentRunLoop runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
}];
[thread start];
[self performSelector:@selector(printInfo) onThread:thread withObject:nil waitUntilDone:NO];
```



## 5.3 小结

梳理本文，比较重要的几个小点如下：

1. `performSelector`内部的实现就是使用`IMP`直接调用对象的方法。
2. 严格遵守ARC规则，注意方法的命名，不然会为内存泄漏埋下伏笔。
3. 反射`NSSelectorFromString()`很强大，但是使用`@selector()`显式告诉编译器将要调用的方法是更优的选择。
4. `performSelector`能实现延迟是因为定时器，要及时取消无法执行的延迟方法。
5. 子线程中使用延迟方法，需要主动使Runloop能够运行。
6. NSTimer在其`invalidate`方法调用后，Runloop会自动移除对它的引用。

具体在业务上，可以举几个简单的场景。

对于文件或者模型的下载，有时候需要用到进度条提示。这时候可以在子线程下载，跑到主线程更新UI：

```
[self performSelectorOnMainThread:@selector(updateProgress) withObject:nil waitUntilDone:NO];
```



有些时候需要进行简单的Tip提示，3秒后自动让它自动隐藏：

```
[self.tipView performSelector:@selector(dismiss) withObject:nil afterDelay:3.f];
```



当使用多态的时候，动态调用方法创建实例对象：

```
Class cls = Nil;
if (item.isForChinese) {
    cls = NSClassFromString(@"YAChinesePopView");
} else {
    cls = NSClassFromString(@"YAEnglishPopView");
}
return [cls performSelector:@selector(popViewWithItem:) withObject:item];
```



偶然间发现一个好玩的例子：直接调用某个对象的某个方法，而不导入其头文件。比如手百自定义一个继承自WKWebView的子类，并把这个实例暴露出来，我们想让它执行某段JavaScript，但是又不想导入繁重的头文件：

```
if ([obj respondsToSelector:@selector(evaluateJavaScript:completionHandler:)]) {
    [obj performSelector:@selector(evaluateJavaScript:completionHandler:) withObject:js withObject:nil];
} 
```

上面的只是冰山一角，其合理性需要结合具体的业务场景具体分析，只是想阐明，performSelector在一些情况下是很有用的。

# 6. Runloop 与NSPort

参考：

1. [**iOS**开发·**RunLoop**源码与用法完全解析**(**输入源，定时源，观察者，线程间通信，端口间通信，**NSPort**，**NSMessagePort**，**NSMachPort**，**NSPortMessage)**](https://www.jianshu.com/p/07313bc6fd24)
2. [**NSPort**线程间通信实例（**Xcode12.3**）](https://juejin.cn/post/6922283273012019213)

#### 6.1 线程间通信的一种方式（另一种是performSelectorOnThread：）

核心思路：

1. 在主线程创建一个NSPort 的实例A 并添加主线程的NSRunLoop中。
2. 再创建一个线程 S 将A作为参数发送到线程中。
3. 线程 S 中创建一个本地NSPort 实例B，也添加到NSRunLoop中。
4. 从B向A发送消息后，将线程S的runloop运行。
5. 主线程收到线程S发送的消息。
6. 主线程向线程S发送消息。
7. 通过 handlePortMessage: 代理方法接受消息。

# 7. **Runloop** 与 **GCD**

- GCD 提供的某些接口用到了 RunLoop， 例如 dispatch_async()。

- 当调用 dispatch_async(dispatch_get_main_queue(), block) 时，libDispatch 会向主线程的 RunLoop 发送消息，RunLoop会被唤醒，并从消息中取得这个 block，并在回调 __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__() 里执行这个 block。但这个逻辑仅限于 dispatch 到主线程，dispatch 到其他线程仍然是由 libDispatch 处理的。

# 8. **Runloop** 与 NSURLConnection

## 8.1 iOS 中，关于网络请求的接口自上至下有如下几层:

-  NSURLSession  -> AFNetworking2、Alamofire  ： NSURLSession 是 iOS7 中新增的接口，表面上是和 NSURLConnection 并列的，但底层仍然用到了 NSURLConnection 的部分功能 (比如 com.apple.NSURLConnectionLoader 线程)，AFNetworking2 和 Alamofire 工作于这一层。

-  NSURLConnection -> AFNetworking  ：  NSURLConnection 是基于 CFNetwork 的更高层的封装，提供面向对象的接口，AFNetworking 工作于这一层。

-  CFNetwork    -> ASIHttpRequest  ：  CFNetwork 是基于 CFSocket 等接口的上层封装，ASIHttpRequest 工作于这一层。

-  CFSocket  ：  CFSocket 是最底层的接口，只负责 socket 通信。

## 8.2 NSURLConnection 的工作流程：

1. 使用 NSURLConnection 时，首先会传入一个 Delegate，然后调用 [connection start] 后，这个 Delegate 就会不停收到事件回调。实际上，start 这个函数的内部会会获取 CurrentRunLoop，然后在其中的 DefaultMode 添加了4个 Source0 (即需要手动触发的Source)：

2. - CFMultiplexerSource 是负责各种 Delegate 回调的
   - CFHTTPCookieStorage 是处理各种 Cookie 的。

2. 当开始网络传输时，可以看到 NSURLConnection 创建了两个新线程：

1. - com.apple.CFSocket.private：处理底层 socket 连接的
   - com.apple.NSURLConnectionLoader：内部会使用 RunLoop 来接收底层 socket 的事件，并通过之前添加的 Source0 通知到上层的 Delegate

3. NSURLConnectionLoader 中的 RunLoop 通过一些基于 mach port 的 Source 接收来自底层 CFSocket 的通知。当收到通知后，其会在合适的时机向 CFMultiplexerSource 等 Source0 发送通知，同时唤醒 Delegate 线程的 RunLoop 来让其处理这些通知。CFMultiplexerSource 会在 Delegate 线程的 RunLoop 对 Delegate 执行实际的回调。

1. 如图：

<img src="/Users/tangh/yuki/博客/文章仓库/YLNoteHub/content/Runloop/image/Runloop_URL_0.png" alt="NSURLConnection工作原理" style="zoom:90%;" />