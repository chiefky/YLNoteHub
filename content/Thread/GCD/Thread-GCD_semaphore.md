#  Dispatch Semaphore

- 标识持有计数的信号。计数小于0 时等待，不可继续执行。计数为 0 或大于 0 时，计数减 1 且不等待，可继续执行

 **Dispatch Semaphore** 提供了三个方法：

- - dispatch_semaphore_create：创建一个 Semaphore 并初始化信号的总量

  - dispatch_semaphore_signal：发送一个信号，让信号总量加 1

  - dispatch_semaphore_wait：可以使总信号量减 1，信号总量小于 0 时就会一直等待（阻塞所在线程），否则就可以正常执行。

  - - 返回值为0，代表成功获取信号量，非0代表超时

<font color='red'>注意：信号量的使用前提是：想清楚你需要处理哪个线程等待（阻塞），又要哪个线程继续执行，然后使用信号量。</font>

**Dispatch Semaphore** 在实际开发中主要用于：

- 保持线程同步，将异步执行任务转换为同步执行任务

​    线程安全：如果你的代码所在的进程中有多个线程在同时运行，而这些线程可能会同时运行这段代码。如果每次运行结果和单线程运行的结果是一样的，而且其他的变量的值也和预期的是一样的，就是线程安全的。

​    🌰代码：

```objective-c
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);

    [self.session getTasksWithCompletionHandler:^(NSArray *dataTasks, NSArray *uploadTasks, NSArray *downloadTasks) {

       dispatch_semaphore_signal(semaphore);

    }];

    // 阻塞当前线程，直到 session 任务完成

    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
```



- 保证线程安全，为线程加锁

   线程同步：可理解为线程 A 和 线程 B 一块配合，A 执行到一定程度时要依靠线程 B 的某个结果，于是停下来，示意 B 运行；B 依言执行，再将结果给 A；A 再继续操作。

​    🌰代码：

```objective-c
    // 创建 信号量 线程锁
    
    dispatch_semaphore_t  semaphoreLock = dispatch_semaphore_create(1);
    
    - (void)saleTicketSafe {

          // 相当于加锁
          dispatch_semaphore_wait(semaphoreLock, DISPATCH_TIME_FOREVER);

          // 需要保证线程安全的操作比如：_name = @"张三"；
          。。。
          // 相当于解锁
          dispatch_semaphore_signal(semaphoreLock);
    }
```



注意：

- dispatch_semaphore_t调用_dispatch_semaphore_dispose 释放时，当前信号量值必须大于等于初始信号量值时，才能正常释放，否则可能引起EXC_BAD_INSTRUCTION指令错误（确实出现过，参考：[**dispatch_semaphore_t** 造成 **EXC_BAD_INSTRUCTION** 崩溃](https://www.jianshu.com/p/792ebd0a79a7)）