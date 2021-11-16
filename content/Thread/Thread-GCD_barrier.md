

# 4. Dispatch Barrier

 当多线程并发读写同一个资源时，为了保证资源读写的正确性，可以用Barrier Block解决该问题。

 Dispatch Barrier会确保队列中先于Barrier Block提交的任务都完成后再执行它，并且执行时队列不会同步执行其它任务，等Barrier Block执行完成后再开始执行其他任务。

## 4.1 dispatch_barrier_async使用

举个🌰：

```objective-c
- (void)testBarrier_create {
    //创建自定义并行队列
    dispatch_queue_t queue = dispatch_queue_create("com.yuli.queue", DISPATCH_QUEUE_CONCURRENT);
    dispatch_async(queue, ^{
        //读操作
        NSLog(@"work1");
    });
    dispatch_barrier_async(queue, ^{
        //barrier block,可用于写操作
        //确保资源更新过程中不会有其他线程读取
        NSLog(@"work2");
        sleep(1);
    });
    dispatch_async(queue, ^{
        //读操作
        NSLog(@"work3");
    });
}
```

<font color='red'>注意点：调用dispatch_barrier_async时,必须提交给一个用DISPATCH_QUEUE_CONCURRENT属性创建的并发队列，不能是系统提供的 global queue。</font>

>  这里有个需要注意也是官方文档上提到的一点，如果我们调用dispatch_barrier_async时将Barrier blocks提交到一个global queue，barrier blocks执行效果与dispatch_async()一致；只有将Barrier blocks提交到使用DISPATCH_QUEUE_CONCURRENT属性创建的并行queue时它才会表现的如同预期。

## 4.2原理





