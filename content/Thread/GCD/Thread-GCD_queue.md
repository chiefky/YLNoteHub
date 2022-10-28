# 1. dispatch_queue使用

## 1.1创建队列

**dispatch_queue_create**()

### 设置队列优先级

**dispatch_queue_attr_make_with_qos_class**

​    🌰代码：

```objective-c
 dispatch_queue_attr_t attr_t = dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_CONCURRENT, QOS_CLASS_USER_INTERACTIVE, QOS_MIN_RELATIVE_PRIORITY);
dispatch_queue_t queue = dispatch_queue_create("com.gcd.thread.yuli", attr_t);
```

`dispatch_queue_attr_make_with_qos_class `参数解读：

- - dispatch_queue_attr_make_with_qos_class 的第二个参数代表队列的优先级，目前优先级有这几个选择：

  - - **QOS_CLASS_USER_INTERACTIVE** （DISPATCH_QUEUE_PRIORITY_HIGH）
    - **QOS_CLASS_USER_INITIATED** （DISPATCH_QUEUE_PRIORITY_HIGH）
    - **QOS_CLASS_UTILITY** （DISPATCH_QUEUE_PRIORITY_LOW）
    - **QOS_CLASS_DEFAULT** （DISPATCH_QUEUE_PRIORITY_DEFAULT）
    - **QOS_CLASS_BACKGROUND** （DISPATCH_QUEUE_PRIORITY_BACKGROUND）

- - dispatch_queue_attr_make_with_qos_class 的第三个参数，需要填写一个负数的偏移值，小于0且大于等于-15(QOS_MIN_RELATIVE_PRIORITY即表示为-15)，必须这么填，不然函数会返回一个null。这个参数主要作用是在你给定的优先级系统不能满足的情况下，如果需要调度的话，给定一个调度偏移值。



## 1.2 设置队列的目标队列：

作用： 变更队列优先级 ， 改变队列层次体系

**dispatch_set_target_queue（q1,q2）**「慎用」

 参考： [**iOS**多线程-dispatch_set_target_queue](https://cloud.tencent.com/developer/article/1411276)



​    🌰代码：

```objective-c
dispatch_queue_t queue1 = dispatch_get_global_queue(NSQualityOfServiceUserInitiated, 0);
dispatch_queue_t queue2 = dispatch_get_global_queue(NSQualityOfServiceUserInteractive, 0);
// 将第二个队列权限设置为第一个队列一样：
dispatch_set_target_queue(queue2, queue1);
```

  

- dispatch_set_target_queue 一共有两个功能，除了 变更队列优先级 外，还可以 改变队列层次体系。当我们想让不同队列中的任务同步的执行时，可以创建一个串行队列，然后将这些队列的target指向新建的队列即可。（经测试，这些任务会在 target 队列，相同的线程里执行）

​	   例如：将多个串行queue指定到目标串行queue, 以实现某任务在多个串行 queue 也是先后执行 而非并行

```objective-c
dispatch_queue_t targetQueue = dispatch_queue_create("test.target.queue", DISPATCH_QUEUE_SERIAL);  
dispatch_queue_t queue1 = dispatch_queue_create("test.1", DISPATCH_QUEUE_SERIAL);  
dispatch_queue_t queue2 = dispatch_queue_create("test.2", DISPATCH_QUEUE_SERIAL);  
dispatch_set_target_queue(queue1, targetQueue);  
dispatch_set_target_queue(queue2, targetQueue);  

dispatch_async(queue1, ^{  
// 会在 target 队列，相同的线程里执行
NSLog(@"1 in");  
[NSThread sleepForTimeInterval:3.f];  
NSLog(@"1 out");  
});  

dispatch_async(queue2, ^{  
// 会在 target 队列，相同的线程里执行
NSLog(@"2 in");  

[NSThread sleepForTimeInterval:2.f];  

NSLog(@"2 out");  

});  
```

输出结果:

> ```objective-c
> 2022-02-27 01:18:01.282229+0800 YLNote[71780:29136276] 1 in <NSThread: 0x600003b64040>{number = 7, name = (null)}
> 2022-02-27 01:18:04.287459+0800 YLNote[71780:29136276] 1 out
> 2022-02-27 01:18:04.287630+0800 YLNote[71780:29136276] 2 in <NSThread: 0x600003b64040>{number = 7, name = (null)}
> 2022-02-27 01:18:06.292643+0800 YLNote[71780:29136276] 2 out
> ```

​     注意：dispatch_set_target_queue设置时机应在设置block任务之前，同时不能互相循环设置

## 1.3 GCD队列标识

### 设置队列标识：dispatch_queue_set_specific

​    🌰代码：

```objective-c
static const void * const kDispatchQueueSpecificKey = &kDispatchQueueSpecificKey;
dispatch_queue_set_specific(queue, kDispatchQueueSpecificKey, &kDispatchQueueSpecificKey, NULL);
```

### 获取队列标识：dispatch_get_specific

   🌰代码：

```objective-c
dispatch_get_specific(kDispatchQueueSpecificKey);
```

## 1.4 GCD**延迟执行：**dispatch_after

- 功能描述：延迟指定时间后，将任务提交到指定队列。具体的任务执行时间，可能会受到当前队列 忙碌状态影响，导致延迟时间更长。具体实现类似：延迟指定时间后调用` dispatch_async(queue1, ^{ });`

   🌰代码：

```objective-c
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1 * NSEC_PER_SEC)), queue, ^{
   NSLog(@"1秒后来到这");
});
```

​	拓展：GCD中除了dispatch_after，还有一种类似于NSTimer的计时器，也就是通过dispatch_source来设置定时器。其实如果看源码dispatch_after内部也是通过dispatch_source进行实现的。

## 1.5 **GCD**一次性执行：**dispatch_once**

- 功能描述：常用于构造一份唯一的实例，用于app内存域中达到资源共享

​	拓展：除了单例这种方式，是否还有别的方式能达到资源共享？在Xcode 8开始，支持了类属性（参考：[**(**译**)Objective-C** 类属性](https://juejin.cn/post/6844903887451734029)）：

​    **@property(nonatomic, copy, class) NSString \*name;**

  所以，我们可以利用这个新的特性，来完成资源共享。

## 1.6 GCD快速迭代方法：**dispatch_apply**（适用于并发队列）

- 功能描述：按照指定的次数将指定的任务追加到指定的队列中，并等待全部队列执行结束。

- - 在串行队列中使用 dispatch_apply，实现效果和 for 循环一样，按顺序同步执行
  - 在并发队列中进行异步操作，dispatch_apply 可以 在多个线程中同时（异步）分别执行

​     注意：无论是在串行队列，还是并发队列中，dispatch_apply 都会等待全部任务执行完毕，这点就像是同步操作，也像是队列组中的 dispatch_group_wait方法



​     🌰代码：

```objective-c
dispatch_queue_attr_t attr_t = dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_CONCURRENT, QOS_CLASS_UTILITY, QOS_MIN_RELATIVE_PRIORITY);

dispatch_queue_t queue = dispatch_queue_create("com.zhaomu.test", attr_t);
dispatch_apply(10, queue, ^(size_t idx) {

  NSLog(@"%@ --> 第%@只熊猫宝宝向你奔来...", [NSThread currentThread],@(idx));

});


```

# 2. 队列的服务质量与优先级

> “**增加并发”在一定范围内可以作为启动优化的方案，但在低端机上，CPU 已经成为瓶颈，并发时异步线程对主线程的抢占也需要引起重视。**
>
> GCD 提供了四种 QoS 给开发者使用，官方也为这四种 QoS 提供了最佳实践建议。
>
> 经过评测和源码推理，User-Interactive 和 User-Initiated 对主线程有明显抢占，Utility 和 Background 对主线程的抢占极少。开发者创建的 GCD 队列，默认的 QoS 实际为 User-Initiated。因此在启动期间（或者任何耗时敏感期间），与启动无直接关系的 queue，应该主动设置为 Utility 或 Background，减少对主线程的抢占。
>
> 通过飞书上落地优化，我们能得出结论：对线程或 GCD queue 调整 QoS，能在不改变启动业务逻辑的情况下取得显著收益。
>
> 当然，比事后优化更好的操作，是在编码时就充分了解不同 QoS 的行为特性，选用最适合的 QoS。
>
> 参考：[不改一行业务代码，飞书 iOS 低端机启动优化实践](https://mp.weixin.qq.com/s/KQJ5QXHdhwHRN65KdD45qA)

队列的服务质量与优先级的对应关系：

```objective-c
__QOS_ENUM(qos_class, unsigned int,
	QOS_CLASS_USER_INTERACTIVE
			__QOS_CLASS_AVAILABLE(macos(10.10), ios(8.0)) = 0x21,
	QOS_CLASS_USER_INITIATED
			__QOS_CLASS_AVAILABLE(macos(10.10), ios(8.0)) = 0x19,
	QOS_CLASS_DEFAULT
			__QOS_CLASS_AVAILABLE(macos(10.10), ios(8.0)) = 0x15,
	QOS_CLASS_UTILITY
			__QOS_CLASS_AVAILABLE(macos(10.10), ios(8.0)) = 0x11,
	QOS_CLASS_BACKGROUND
			__QOS_CLASS_AVAILABLE(macos(10.10), ios(8.0)) = 0x09,
	QOS_CLASS_UNSPECIFIED
			__QOS_CLASS_AVAILABLE(macos(10.10), ios(8.0)) = 0x00,
);

```

| 优先级 | 新版(iOS8以后)                                               | 旧版本                             |
| ------ | ------------------------------------------------------------ | ---------------------------------- |
| 最高   | QOS_CLASS_USER_INTERACTIVE<br /> //用户交互权限，属于最高等级，常被用于处理交互事件或者刷新UI，因为这些需要即时的 | Main Thread                        |
| 高     | QOS_CLASS_USER_INITIATED<br />  // 由用户发起的并且需要立即得到结果的任务，比如滑动 scroll view 时去加载数据用于后续 cell 的显示，这些任务通常跟后续的用户交互相关，在几秒或者更短的时间内完成； | DISPATCH_QUEUE_PRIORITY_HIGH       |
| 默认   | QOS_CLASS_DEFAULT<br />  // 默认权限，具体权限由系统根据实际情况来决定使用哪个等级权限，如果实际情况不太利于决定使用何种权限，则从UserInitiated和Utility之间选一个权限并使用 | DISPATCH_QUEUE_PRIORITY_DEFAULT    |
| 低     | QOS_CLASS_UTILITY<br />  // 不需要马上就能得到结果，比如下载任务。当资源被限制后，此权限的任务将运行在节能模式下以提供更多资源给更高的优先级任务 | DISPATCH_QUEUE_PRIORITY_LOW        |
| 后台   | QOS_CLASS_BACKGROUND<br />  // 后台权限，通常用户都不能意识到有任务正在进行，比如数据备份等。大多数处于节能模式下，需要把资源让出来给更高的优先级任务 | DISPATCH_QUEUE_PRIORITY_BACKGROUND |

## 2.1  dispatch_queue_attr_make_with_qos_class

作用： **初始化队列时，设置队列服务质量**

```objective-c
/// 设置队列优先级
- (void)testQueue_qos_make {
    dispatch_queue_attr_t arr = dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_CONCURRENT, QOS_CLASS_USER_INTERACTIVE, QOS_MIN_RELATIVE_PRIORITY);
    dispatch_queue_t queue = dispatch_queue_create("yuli.gcd.queue.concurrent", arr);
    dispatch_async(queue, ^{
        NSLog(@"%s say: Hello world!",dispatch_queue_get_label(queue));
    });
}
```

## 2.2 dispatch_set_target_queue

作用： **变更队列优先级or改变队列层次体系**

### 2.2.1 变更队列优先级

```objective-c
- (void)testQueue_set_target {
  dispatch_queue_t q1 = dispatch_queue_create("q1", DISPATCH_QUEUE_SERIAL);
  dispatch_queue_t globalQueue = dispatch_get_global_queue(QOS_CLASS_USER_INTERACTIVE, 0);
  NSLog(@"before (%s: %u,%s:%u)",dispatch_queue_get_label(q1),dispatch_queue_get_qos_class(q1, nil),dispatch_queue_get_label(globalQueue),dispatch_queue_get_qos_class(globalQueue, nil));

  dispatch_set_target_queue(q1, globalQueue);
  dispatch_async(q1, ^{
      NSLog(@"1");
  });
  NSLog(@"after (%s: %u,%s:%u)",dispatch_queue_get_label(q1),dispatch_queue_get_qos_class(q1, nil),dispatch_queue_get_label(globalQueue),dispatch_queue_get_qos_class(globalQueue, nil));
}
```

### 2.2.2 改变队列层次体系

```objective-c
/// 目标队列可以成为原队列的执行阶层(✅：验证通过)
- (void)testQueue_set_target1 {
    dispatch_queue_t q1 = dispatch_queue_create("q1", DISPATCH_QUEUE_SERIAL);
    dispatch_queue_t q2 = dispatch_queue_create("q2", DISPATCH_QUEUE_SERIAL);
    dispatch_queue_t q3 = dispatch_queue_create("q3", DISPATCH_QUEUE_SERIAL);
    dispatch_queue_t targetQueue = dispatch_queue_create("serial", DISPATCH_QUEUE_SERIAL);
    dispatch_set_target_queue(q1, targetQueue);
    dispatch_set_target_queue(q2, targetQueue);
    dispatch_set_target_queue(q3, targetQueue);
    dispatch_async(q1, ^{
        NSLog(@"1");
    });
    
    dispatch_async(q2, ^{
        NSLog(@"2");
    });
    
    dispatch_async(q3, ^{
        NSLog(@"3");
    });
}

```

# 3. 相关面试题

## 3.1  **GCD** 队列的挂起与恢复

#### 1.为什么挂起与恢复"只适用于自定义的队列，不适用于**dispatch_get_global_queue**等" ？

答：Dispatch全局并发队列在系统管理的线程池之上提供优先级存储桶。系统将根据需求和系统负载决定分配给该池的线程数量。具体地说，系统会尝试为该资源保持良好的并发级别，并在系统调用中阻塞太多现有工作线程时创建新线程。

全局并发队列是共享资源，因此，此资源的每个用户都有责任不向此池提交无限量的工作，特别是可能阻塞的工作，因为这可能会导致系统产生非常大量的线程(又名。线爆炸)。

提交到全局并发队列的工作项在提交顺序方面没有排序保证，提交到这些队列的工作项可以并发调用。

Dispatch全局并发队列是DISPATCH_GET_GLOBAL_QUEUE()返回的众所周知的全局对象。这些对象不能修改。调用DISPATCH_SUSPEND()、DISPATCH_RESUME()、DISPATCH_SET_CONTEXT()等 **在`dispatch_get_global_queue`队列中使用时不起作用**。





