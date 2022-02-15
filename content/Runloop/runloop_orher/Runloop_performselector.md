# Runloop与performSelector

参考：

1. [**iOS**源码解析**: performSelector**是如何实现的？](https://juejin.cn/post/6844904115185664008#heading-16)
2. [PerformSelector原理](https://blog.chenyalun.com/2018/09/30/PerformSelector原理/#内存泄露)

## 1. 接口

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

## 2. 问题Q&A

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



## 小结

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