# 保活线程有几种方式

**前情提要：**

* 进入休眠的 run loop 仅能通过 mach port 和 mach_msg 来唤醒;
* CFRunLoopWakeUp 函数内部是通过 run loop 实例的 _wakeUpPort 成员变量来唤醒 run loop 的;

### 1. 添加事件源（source0/source1）

#### CFRunLoopSourceRef底层源码分析

```c
typedef struct __CFRunLoopSource * CFRunLoopSourceRef;

struct __CFRunLoopSource {
    CFRuntimeBase _base;
    uint32_t _bits;
    pthread_mutex_t _lock;
    CFIndex _order;			/* immutable */
    CFMutableBagRef _runLoops;
    union {
      CFRunLoopSourceContext version0;	/* immutable, except invalidation */
      CFRunLoopSourceContext1 version1;	/* immutable, except invalidation */
    } _context;
};
```

#### source0 和 source1 的区别

<img src="/Users/tangh/yuki/博客/文章仓库/YLNoteHub/content/images/runloop_CFRunLoopSourceContext.png" alt="runloop_CFRunLoopSourceContext" style="zoom:60%;" />



Source0 解读：

* source0 仅包含一个回调函数（perform），它并不能主动唤醒 run loop（进入休眠的 run loop 仅能通过 mach port 和 mach_msg 来唤醒）。
* 你需要先调用 CFRunLoopSourceSignal(rls) 将这个 source 标记为待处理，然后手动调用 CFRunLoopWakeUp(rl) 来唤醒 run loop（CFRunLoopWakeUp 函数内部是通过 run loop 实例的 _wakeUpPort 成员变量来唤醒 run loop 的）
* 唤醒后的 run loop 继续执行 __CFRunLoopRun 函数内部的外层 do while 循环来执行 timers（执行到达执行时间点的 timer 以及更新下次最近的时间点） 和 sources 以及 observer 回调 run loop 状态，其中通过调用 __CFRunLoopDoSources0 函数来执行 source0 事件，执行过后的 source0 会被 __CFRunLoopSourceUnsetSignaled(rls) 标记为已处理，后续 run loop 循环中不会再执行标记为已处理的 source0。
* source0 不同于不重复执行的 timer 和 run loop 的 block 链表中的 block 节点，source0 执行过后不会自己主动移除，不重复执行的 timer 和 block 执行过后会自己主动移除，执行过后的 source0 可手动调用 CFRunLoopRemoveSource(CFRunLoopGetCurrent(), rls, kCFRunLoopDefaultMode) 来移除。

参考链接：https://juejin.cn/post/6913094534037504014

####  添加source0 保活线程

```objective-c
#pragma mark - 手动终止runloop
- (void)stopLoop {
    CFRunLoopRef cf = [NSRunLoop currentRunLoop].getCFRunLoop;
    NSLog(@"-：CFRunLoopStop()终止当前线程的runloop");
    CFRunLoopStop(cf);
}
```

```objc
#pragma mark - 自定义source
/// 终止子线程的runloop
- (void)testSource0_stop {
    [self performSelector:@selector(stopLoop) onThread:self.myThread withObject:nil waitUntilDone:YES];
}

/// 开启子线程
- (void)testSource0 {
    YLThread *thread = [[YLThread alloc] initWithTarget:self selector:@selector(taskSource0) object:nil];
    self.myThread = thread;
    thread.name = @"YLThread.source0";
    [thread start];
}

/// 子线程任务
- (void)taskSource0 {
    CFRunLoopSourceContext  context = {0, (__bridge void *)(self), NULL, NULL, NULL, NULL, NULL,NULL,NULL,RunLoopSourcePerformRoutine};
    // 创建&添加事件源source0
    CFRunLoopSourceRef source0 = CFRunLoopSourceCreate(NULL, 0, &context);
    CFRunLoopAddSource(CFRunLoopGetCurrent(), source0, kCFRunLoopDefaultMode);
    NSLog(@"🍑 runloop start：%@",[NSThread currentThread].name);
    // 将source0任务标记为待处理事件
    CFRunLoopSourceSignal(source0);
    CFRunLoopRun(); //这里启动runloop可以参考下面source1的五种方式，以及是否能成功终止
    NSLog(@"🍑 runloop finished：%@",[NSThread currentThread].name);
}
```

#### 添加source1保活线程

* 在 Cocoa Foundation 中，我们根本不需要直接创建 source1，只需创建一个端口对象，并使用 NSRunLoop  的实例方法将该端口添加到 run loop 中。port 对象会处理所需 source1 的创建和配置。如下代码在子线程中:

  ```cpp
  NSPort *port = [NSPort port];
  [[NSRunLoop currentRunLoop] addPort:port forMode:NSDefaultRunLoopMode];		
  ```

* 在 Core Foundation 中则必须手动创建端口及其 source1。

使用举例🌰：

<img src="/Users/tangh/yuki/博客/文章仓库/YLNoteHub/content/images/runloop_usage_source1.png" alt="runloop_usage_source1" style="zoom:60%;" />



附源码：

```objective-c
#pragma mark- mach port （Foundation 下只能通过port自动完成source1的创建和配置；CoreFoundation下则需要手动创建port和source1
/// 开启子线程
- (void)testMachPort {
    YLThread *thread = [[YLThread alloc] initWithTarget:self selector:@selector(taskPort) object:nil];
    self.myThread = thread;
    thread.name = @"YLThread.mach_port";
    [thread start];
}

/// 终止子线程runloop
- (void)testMachPort_stop {
    [self performSelector:@selector(stopLoop) onThread:self.myThread withObject:nil waitUntilDone:YES];
}

/// 子线程任务（备注：❌：代表无法手动终止runloop（原因是：会无限执行一个无事件源的runMode方法）； ✅：代表可以手动终止runloop）
- (void)taskPort {
    NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
    [runLoop addPort:[NSPort port] forMode:NSDefaultRunLoopMode];
    NSLog(@"🍎 runloop start：%@", [NSThread currentThread].name);
    NSLog(@"do things what you want");
    // 方式1：使用Foundation接口开启runloop
    //            [runLoop run]; // run: ❌
    //            [runLoop runUntilDate:[NSDate distantFuture]]; // runUntilDate: ❌
    [runLoop runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]]; // runMode: ✅
    
    // 方式2：使用Core Foundation接口开启runloop
    //            CFRunLoopRun(); // CFRunLoopRun: ✅
    //            CFRunLoopRunInMode(kCFRunLoopDefaultMode, 1000000, YES); // CFRunLoopRunInMode: ✅
    NSLog(@"🍎 finished： %@",[NSThread currentThread].name);
}
```

### 2. 添加时间源（timer）

根据timer初始化方式不同分几种情况如下：

* 以下方式会自动创建一个timer加入**<font color='red'>已开启</font>**的runloop中（注意：只是添加进runloop，并没有run起来，还是必须得手动run才行）

  * `scheduledTimerWithTimeInterval:(NSTimeInterval)interval repeats:(BOOL)repeats block:(void (^)(NSTimer *timer))block;`
  * `scheduledTimerWithTimeInterval:(NSTimeInterval)ti target:(id)aTarget selector:(SEL)aSelector userInfo:(nullable id)userInfo repeats:(BOOL)yesOrNo;`
  * `scheduledTimerWithTimeInterval:(NSTimeInterval)ti invocation:(NSInvocation *)invocation repeats:(BOOL)yesOrNo;`
  * `performSelector:(SEL)aSelector withObject:(nullable id)anArgument afterDelay:(NSTimeInterval)delay;`
  * `performSelector:(SEL)aSelector withObject:(nullable id)anArgument afterDelay:(NSTimeInterval)delay inModes:(NSArray<NSRunLoopMode> *)modes;`

  注意：子线程的runloop默认是不开启的，需要手动`run`起来，以上接口的timer才会被自动加入到runloop中；

* 需要手动添加入runloop中

  * `timerWithTimeInterval:(NSTimeInterval)ti invocation:(NSInvocation *)invocation repeats:(BOOL)yesOrNo;`
  * `timerWithTimeInterval:(NSTimeInterval)ti target:(id)aTarget selector:(SEL)aSelector userInfo:(nullable id)userInfo repeats:(BOOL)yesOrNo;`
  * `timerWithTimeInterval:(NSTimeInterval)interval repeats:(BOOL)repeats block:(void (^)(NSTimer *timer))block;`
  * `initWithFireDate:(NSDate *)date interval:(NSTimeInterval)interval repeats:(BOOL)repeats block:(void (^)(NSTimer *timer))block;`
  * `initWithFireDate:(NSDate *)date interval:(NSTimeInterval)ti target:(id)t selector:(SEL)s userInfo:(nullable id)ui repeats:(BOOL)rep ;`

  以上timer需要`[NSRunLoop addTimer:]` 手动加入runloop才可；

  

  <font color='red'> 有个疑问，为什么在线程外部终止timer runloop不会退出？</font>

  答：[timer invalid]之后，猜测是，timer终止但是并没有成功从runloop中移除，runloop直到limitDate都不会销毁。

  另一种情况如果是通过`run`或`runUntilData:futrue`开启的runloop，无法终止是因为会无限重复start->exit这个过程（因为这个runloop中没有事件源也没有timer），并不是没有退出，而是每次runloop都是开启后立即退出！！！

实例：

<img src="/Users/tangh/yuki/博客/文章仓库/YLNoteHub/content/images/runloop_usage_timer1.png" alt="runloop_usage_timer" style="zoom:50%;" />

<font color='red'>重点结论：当前时刻未超过多个 timerInterval 时，timer 触发只会延迟执行，不会丢失。若超过多个 timerInterval 时，只会执行 最早应该触发的那次 timer，它之后的 与 当前时刻之间的其他 触发时机 都会舍弃掉</font>

