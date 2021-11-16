# Dispatch Block

GCD 中的任务有两种封装：dispatch_block_t 和 dispatch_function_t，且 dispatch_block_t 比较常用。

dispatch_block，可以理解为一个block对象，拿到这个对象可以让我们更灵活的操作一个任务，比如等待、执行以及监听任务的完成等.

## 1. dispatch_block_create函数

- dispatch_block_create
- dispatch_block_create_with_qos_class

 🌰代码：

```
dispatch_block_t block = ^{
  NSLog(@"一只熊脑宝宝向你奔来...");
};  
```

 一般情况下，按照上面🌰代码的操作，block 是创建在栈上的，通过 **dispatch_block_create** 方法可以使block 创建在堆上。

```
dispatch_block_t  dispatch_block_create(dispatch_block_flags_t flags, dispatch_block_t block);
dispatch_block_t  dispatch_block_create_with_qos_class(dispatch_block_flags_t flags, dispatch_qos_class_t qos_class, int relative_priority, dispatch_block_t block);
```

此方法相比于`dispatch_block_create`多了一个`dispatch_qos_class_t`属性，用来设置优先级；以及`relative_priority`属性，表示偏移值，这个参数主要作用是在你给定的优先级系统不能满足的情况下，如果需要调度的话，给定一个调度偏移值。

### 1.1 `dispatch_block_flags_t`枚举值解释

![dispatch_block_flags_t源码](file:///Users/tangh/yuki/%E5%8D%9A%E5%AE%A2/%E6%96%87%E7%AB%A0%E4%BB%93%E5%BA%93/YLNoteHub/content/Thread/image/Thread_GCD_2_0.png?lastModify=1637039966)

 关于`dispatch_block_flags_t`标志位 枚举值介绍：

**DISPATCH_BLOCK_BARRIER**： 当提交到`DISPATCH_QUEUE_CONCURRENT`队列，类似于`dispatch_barrier_async`（后面会介绍）作用。如果标记为这个的block对象被直接调用，将没有barrier效果。

 **DISPATCH_BLOCK_DETACHED**：block对象将解除与当前执行上下文属性的关联，如果直接调用，在分配给block属性之前，在调用线程上block对象将在block任务执行期间移出这些属性。如果提交到队列，block对象将使用队列属性或者分配给Block对象的属性。【注：DETACHED *[dɪ'tætʃt]* ：单独的 / 分离的 / 超然的 / 独立的】

**DISPATCH_BLOCK_ASSIGN_CURRENT**：Block对象被创建的同时会为block对象分配执行上下文属性。如果直接调用，block对象将在block任务执行期间将这些属性应用于调用线程。如果block任务被提交到队列，则这个标识将在提交队列的同时会替换其所关联的block对象默认的上下文属性。

**DISPATCH_BLOCK_NO_QOS_CLASS**： 表示不能设置优先级属性给block，如果block被直接调用，将会使用当前线程的优先级。如果被提交到队列，在提交到队列的同时将会取消原来的优先级属性。在dispatch_block_create_with_qos_class函数中，这个属性无效。

**DISPATCH_BLOCK_INHERIT_QOS_CLASS**： block和队列同时有优先级属性的情况下，优先使用队列的优先级。当队列没有优先级属性的情况下，block的优先级才会被采用，当block被执行调用，这个属性无效；如果被提交到并行异步队列，这个属性是默认的。【 **注**：INHERIT *[ɪn'herɪt]* ：继承/ 遗传 / 接手 】

 **DISPATCH_BLOCK_ENFORCE_QOS_CLASS**： block的优先级属性要高于队列的优先级属性。如果block被直接调用或被提交到并行同步队列，这个属性是默认的。 【 **注**：ENFORCE  *[ɪn'fɔːrs]*：实施 / 强制履行 / 强迫 】

### 1.2 关于dispatch_block_flags_t 标志位 的使用规则：

- 创建出来的 block 提交到队列的时候同时会为block 赋值一个默认的优先级属性，但也有例外，这三个标识位就不会默认设置优先级，分别是 DISPATCH_BLOCK_ASSIGN_CURRENT、DISPATCH_BLOCK_NO_QOS_CLASS和DISPATCH_BLOCK_DETACHED。
- 当 block 放入并行同步队列，默认是 DISPATCH_BLOCK_ENFORCE_QOS_CLASS 

- 当 block 放入并行异步队列，默认是 DISPATCH_BLOCK_INHERIT_QOS_CLASS
- 如果一个被赋值了优先级属性的block对象被放入到一个串行队列，那么系统将会尽可能的让已经在前面的block对象与这个block对象拥有一个优先级或者更高优先级，以让前面的block任务优先执行。

## 2. dispatch_block_notify函数

作用：**在被观察块 block1 执行完毕之后，立即将通知块 block2 提交到指定队列。**

```
/*!
 * @param block               block1, 需要观察的block
 * @param queue               notification_block提交的队列
 * @param notification_block  block2, 需要通知的block
 */
void dispatch_block_notify(dispatch_block_t block, dispatch_queue_t queue, dispatch_block_t notification_block);
```

举个🌰：

```
/// block1执行完后将block2加入队列
- (void)testBlock_notify {
    dispatch_queue_t queue = dispatch_queue_create("queue", DISPATCH_QUEUE_SERIAL);
    dispatch_block_t block1 = dispatch_block_create(0, ^{
        NSLog(@"block1 begin");
        [NSThread sleepForTimeInterval:1];
        NSLog(@"block1 done");
    });
    dispatch_async(queue, block1);
    dispatch_block_t block2 = dispatch_block_create(0, ^{
        NSLog(@"block2 start,queue");
    });
    //当block1执行完毕后，提交block2到global queue中执行
    dispatch_block_notify(block1, dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), block2);
}
```

## 3. dispatch_block_wait函数

作用： **同步等待，直到指定的 block 执行完成或指定的超时时间结束为止才返回；**  设置等待时间 DISPATCH_TIME_NOW 会立刻返回，  设置 DISPATCH_TIME_FOREVER 会无限期等待指定的 block 执行完成才返回。

```
/*!
 * @param block
 *
 * @param timeout 超时时长
 * 
 * @return long  如果 block 在指定的超时时间内完成，则返回0； 超时则返回非0。
 */
long dispatch_block_wait(dispatch_block_t block, dispatch_time_t timeout);
```

举个🌰：

```
/**
 同步等待，直到指定的 block 执行完成或指定的超时时间结束为止才返回；
 设置等待时间 DISPATCH_TIME_NOW 会立刻返回，
 设置 DISPATCH_TIME_FOREVER 会无限期等待指定的 block 执行完成才返回。
 */
- (void)testBlock_wait {
    dispatch_queue_t queue = dispatch_queue_create("queue", DISPATCH_QUEUE_SERIAL);
    dispatch_block_t block = dispatch_block_create(0, ^{
        NSLog(@"begin %@",[NSThread currentThread]);
        [NSThread sleepForTimeInterval:5];
        NSLog(@"end %@",[NSThread currentThread]);
    });
    dispatch_async(queue, block);
    //等待前面的任务执行完毕
    dispatch_time_t interval = DISPATCH_TIME_FOREVER; // dispatch_time(DISPATCH_TIME_NOW, 4);
   long result = dispatch_block_wait(block, interval);
    NSLog(@"coutinue res:%ld %@",result,[NSThread currentThread]);
}
```

## 4. dispatch_block_cancel函数

```
void dispatch_block_cancel(dispatch_block_t block); 
```

举个🌰：

```
/// 异步取消指定的 block，正在执行的 block 不会被取消。
- (void)testBlock_cancel {
    dispatch_queue_t queue = dispatch_queue_create("queue", DISPATCH_QUEUE_SERIAL);
    dispatch_block_t block1 = dispatch_block_create(0, ^{
        NSLog(@"block1 begin");
        [NSThread sleepForTimeInterval:5];
        NSLog(@"block1 done");
    });
    dispatch_block_t block2 = dispatch_block_create(0, ^{
        NSLog(@"block2");
    });
    dispatch_block_t block3 = dispatch_block_create(0, ^{
        NSLog(@"block3");
    });

    dispatch_async(queue, block1);
    dispatch_async(queue, block2);
    dispatch_async(queue, block3);
    //取消block2
    dispatch_block_cancel(block2);
    //测试block2是否被取消 ,"测试指定的 block 是否被取消。返回非0代表已被取消；返回0代表没有取消"
    NSLog(@"block2是否被取消:%ld",dispatch_block_testcancel(block2));
}
```