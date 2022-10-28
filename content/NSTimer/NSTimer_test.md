# 1、iOS 常用的计时器有哪几种，一般在什么场景下使用？

- NSTimer和CADisplayLink依赖于RunLoop，如果RunLoop的任务过于繁重，可能会导致NSTimer不准时，相比之下GCD的定时器会更加准时，因为GCD不是依赖RunLoop，而是由内核决定
- CADisplayLink和NSTimer会对target产生强引用，如果target又对它们产生强引用，那么就会引发循环引用



## 1.1 NSTimer

> 使用场景：重复执行某种操作

## 1.2 CADisplayLink

> 使用场景：从原理上可以看出，CADisplayLink适合做界面的不停重绘，比如视频播放的时候需要不停地获取下一帧用于界面渲染。

## 1.2 DispatchSourceTimer

> - 最小精度为纳秒,误差在50毫秒以内.
> - 比较常用的 dispatch_after方法并没有直接的cancel方法

# 2、timer 怎么解决循环引用？

> 1. 自定义timer，使timer的target不直接使用vc；
>    注意：自定义timer与NSTimer间仍然存在强引用，需要在vc dealloc之前stop timer
>
> 2. 使用iOS10新增的`scheldueTimerWithBlock:`;
>    注意：
>
>    * block仍然会产生循环引用，block内使用weakSelf
>    * 持用 NSTimer 对象的类的方法中 -(void)dealloc 调用 NSTimer  的- (void)invalidate 方法；
>
> 3. 使用block，类似iOS新增的接口
>
>    ```objective-c
>    @implementation NSTimer (SafeBlock)
>    
>    + (instancetype)yl_ScheduledTimerWithTimeInterval:(NSTimeInterval)timeInterval repeats:(BOOL)repeats block:(void (^)(void))block {
>        return [NSTimer scheduledTimerWithTimeInterval:timeInterval target:self selector:@selector(handle:) userInfo:[block copy] repeats:repeats];
>    }
>    
>    + (void)handle:(NSTimer *)timer {
>        void(^block)(void) = timer.userInfo;
>        if (block) {
>            block();
>        }
>    }
>    
>    @end
>    ```
>
>    注意：同2
>
> 4. NSProxy虚基类的方式
>    ///使用NSProxy（其实是借助参数传递中间者+中间者弱持有self就可以解决，完全可以不用消息转发）（使用Proxy的原理是：1.添加了一个中间者Proxy；2.Proxy持有一个弱引用对象，也就是响应方法的目标对象；3. 借助消息转发机制将消息传递给目标对象）
>
>    注意：
>
>    *  持用 NSTimer 对象的类的方法中 -(void)dealloc 调用 NSTimer  的- (void)invalidate 方法；

# 3、NSTimer 都有哪些情况会不准？

> 1. defaultMode、trackingMode
> 2. Runloop中调用了耗时时间长的任务，任务同步

# 