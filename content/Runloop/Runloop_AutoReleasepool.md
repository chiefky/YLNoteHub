**Runloop** 与 **AutoReleasePool**



1、主线程会监听 runloop 运行状态，在 runloop 的 entry(即将进入runllop)状态创建 autoreleasepool 并在 BeforeWaiting(即将进入休眠) 状态进行释放。具体过程如下：

- App启动后，苹果在主线程 RunLoop 里注册了两个 Observer，其回调都是 _wrapRunLoopWithAutoreleasePoolHandler()。



- - 第一个 Observer 监视的事件是 Entry(即将进入Loop)，其回调内会调用 _objc_autoreleasePoolPush() 创建自动释放池。其 order 是-2147483647，优先级最高，保证创建释放池发生在其他所有回调之前。



- - 第二个 Observer 监视了两个事件： BeforeWaiting(准备进入休眠) 时调用_objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush() 释放旧的池并创建新池；Exit(即将退出Loop) 时调用 _objc_autoreleasePoolPop() 来释放自动释放池。这个 Observer 的 order 是 2147483647，优先级最低，保证其释放池子发生在其他所有回调之后。



2、子线程会在创建的时候，同时创建一个autoreleasepool ，然后在线程kill 掉的时候释放掉

3、main方法为什么会包一层 autoreleasePool？

为了释放在 UIApplicationMain 方法调用后，到主线程 runloop 创建 autoreleasepool 之前的对象，也就是 NSStringFromClass([AppDelegate class]) 对象。