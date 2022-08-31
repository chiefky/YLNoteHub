# 1、NSTimer创建方式

```objective-c
//    第一种：
    self.myTimer = [NSTimer timerWithTimeInterval:1
                                         target:self
                                         selector:@selector(printLog:)
                                       userInfo:nil
                                        repeats:YES];
    [[NSRunLoop currentRunLoop] addTimer:self.myTimer
                                      forMode:NSRunLoopCommonModes];

    // 第二种：
    self.myTimer = [NSTimer scheduledTimerWithTimeInterval:1
                                                  target:self
                                                selector:@selector(printLog)
                                                userInfo:nil
                                                 repeats:YES];

    // 第三种：iOS10以上使用，将target的循环引用变更为block的循环引用,使用时注意打破block的循环引用
    __weak typeof(self) weakSelf = self;
    self.myTimer = [NSTimer scheduledTimerWithTimeInterval:1 repeats:YES block:^(NSTimer * _Nonnull timer) {
        __strong typeof(weakSelf) strongSelf = weakSelf;
        [strongSelf printLog];
    }];
```



# 2、NSTimer怎么解决循环引用？

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