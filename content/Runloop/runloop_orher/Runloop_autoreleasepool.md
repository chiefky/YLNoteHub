## 5.1 Runloop 与 AutoReleasePool

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

### 5.1.1 主线程的Autoreleasepool与子线程的Autoreleasepool

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

### 5.1.2 main方法为什么会包一层 autoreleasePool？

为了释放 "主线程 runloop 创建 autoreleasepool 之前的对象", 也就是 NSStringFromClass([AppDelegate class]) 内部创建的autorelease对象。