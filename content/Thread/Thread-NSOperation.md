#  1. NSOperation与GCD的区别

> 区别：
>
> - **1.** GCD 的核心是 C 语言写的系统服务，执行和操作简单高效，因此 NSOperation 底层也通过 GCD 实现，换个说法就是 NSOperation 是对 GCD 更高层次的抽象，这是他们之间最本质的区别。因此如果希望自定义任务，建议使用 NSOperation；
> - **2.** 依赖关系，NSOperation 可以设置两个 NSOperation 之间的依赖，第二个任务依赖于第一个任务完成执行，GCD 无法设置依赖关系，不过可以通过dispatch_barrier_async来实现这种效果；
> - **3.** KVO(键值对观察)，NSOperation 和容易判断 Operation 当前的状态(是否执行，是否取消)，对此 GCD 无法通过 KVO 进行判断；
> - **4.** 优先级，NSOperation 可以设置自身的优先级，但是优先级高的不一定先执行，GCD 只能设置队列的优先级，无法在执行的 block 设置优先级；
> - **5.** 继承，NSOperation 是一个抽象类，实际开发中常用的两个类是 NSInvocationOperation 和 NSBlockOperation ，同样我们可以自定义 NSOperation，GCD 执行任务可以自由组装，没有继承那么高的代码复用度；
> - **6.** 效率，直接使用 GCD 效率确实会更高效，NSOperation 会多一点开销，但是通过 NSOperation 可以获得依赖，优先级，继承，键值对观察这些优势，相对于多的那么一点开销确实很划算，鱼和熊掌不可得兼，取舍在于开发者自己；

Operation Queues 是一个建立在 GCD 的基础之上的，面向对象的解决方案。它使用起来比 GCD 更加灵活，功能也更加强大。

下面简单地介绍 Operation Queues 和 GCD 各自的使用场景：

- Operation Queues ：相对 GCD 来说，使用 Operation Queues 会增加一点点额外的开销，但是我们却换来了非常强大的灵活性和功能，我们可以给 operation 之间添加依赖关系、取消一个正在执行的 operation 、暂停和恢复 operation queue 等；

- GCD ：则是一种更轻量级的，以 FIFO 的顺序执行并发任务的方式，使用 GCD 时我们并不关心任务的调度情况，而让系统帮我们自动处理。但是 GCD 的短板也是非常明显的，比如我们想要给任务之间添加依赖关系、取消或者暂停一个正在执行的任务时就会变得非常棘手。

# 2. NSOperation 

在 iOS 开发中，我们可以使用 NSOperation 类来封装需要执行的任务，而一个 operation 对象（以下正文简称 operation ）指的就是 NSOperation 类的一个具体实例。**NSOperation 本身是一个抽象类，不能直接实例化，**因此，如果我们想要使用它来执行具体任务的话，就必须创建自己的子类或者使用系统预定义的两个子类，NSInvocationOperation 和 NSBlockOperation 。

另外，所有的 operation 都支持以下特性：

- 支持在 operation 之间建立依赖关系，只有当一个 operation 所依赖的所有 operation 都执行完成时，这个 operation才能开始执行；
- 支持一个可选的 completion block ，这个 block 将会在 operation 的主任务执行完成时被调用；
- 支持通过 `KVO` 来观察 operation 执行状态的变化；
- 支持设置执行的优先级，从而影响 operation 之间的相对执行顺序；
- 支持取消操作，可以允许我们停止正在执行的 operation 。



> **NSOperation** 抽象类的属性与功能描述：
>
> - ~~@property (readonly, getter=isConcurrent) BOOL concurrent~~  ：// To be deprecated; use and override 'asynchronous' below
>
> - @property (readonly, getter=isAsynchronous) BOOL asynchronous：
>
> - -  NSOperation 类的 isAsynchronous 方法的返回值标识了一个 operation 相对于调用它的 start 方法的线程来说是否是异步执行的。
>   - 在默认情况下，isAsynchronous 方法的返回值是 NO ，也就是说 它会在当前线程下同步执行。
>
> - @property (nullable, copy) void (^completionBlock)(void)：
>
> - - operation 在它的任务执行完成时回调的 block。
>   - 即使在 **isAsynchronous = NO** 的情况下，**completionBlock** 的调用时机也在 **start** 方法之后，且是异步的。
>   - 当一个 **operation** 被取消时，它的 **completion block** 仍然会执行
>   - 无法保证 **completion block** 一定在主线程，理论上它应该是与触发 **isFinished** 的 **KVO** 通知所在的线程一致的。所以如果有必要的话我们可以在 **completion block** 中使用 **GCD** 来保证从主线程更新 **UI** 。
>
> - 配置依赖关系（第一优先级，通过依赖关系可以决定 operation 的 isReady 状态）：
>
> - - \- (void)addDependency:(NSOperation *)op：添加依赖，使当前操作依赖于操作 op 的完成
>   - \- (void)removeDependency:(NSOperation *)op：移除依赖，取消当前操作对操作 op 的依赖
>   - @property (readonly, copy) NSArray<NSOperation *> *dependencies：在当前操作开始执行之前，完成执行的所有操作对象数组。
>   - 1. 1. 注意：用 addDependency: 方法添加的依赖关系是单向的，比如 [A addDependency:B]; ，表示 A 依赖 B，B 并不依赖 A 。
>        2. 注意：这里的依赖关系并不局限于相同 operation queue 中的 operation 之间，配置依赖关系的方法是存在于 NSOperation 类中的，operation 的依赖关系是它自己管理的，与它被添加到哪个 operation queue 无关。完全可以给一些 operation 配置好依赖关系，然后将它们添加到不同的 operation queue 中。
>        3. 注意：不要在 operation 之间添加循环依赖，因为这样会导致这些 operation 都不会被执行。
>        4. 注意：我们应该在手动执行一个 operation 或将它添加到 operation queue 前配置好依赖关系，因为在之后添加的依赖关系可能会失效。
>
> - **@property NSOperationQueuePriority queuePriority**；修改 **Operation** 在队列中的优先级（优先级低于 isReady 也就是依赖关系）
>
> - - operation 的 isReady 状态取决于它的依赖关系，而在队列中的优先级则是 operation 本身的属性。
>   - 默认情况下，所有新创建的 operation 的队列优先级都是 normal 的
>   - 可以根据需要通过 setQueuePriority: 方法来提高或降低 operation 的队列优先级：
>
> ​       typedef NS_ENUM(NSInteger, NSOperationQueuePriority) {
>
> ​			NSOperationQueuePriorityVeryLow = -8L,
>
> ​			NSOperationQueuePriorityLow = -4L,
>
> ​			NSOperationQueuePriorityNormal = 0,
>
> ​			NSOperationQueuePriorityHigh = 4,
>
> ​			NSOperationQueuePriorityVeryHigh = 8
>
> ​	   };
>
> - - 注意：队列优先级只应用于相同 operation queue 中的 operation 之间，不同 operation queue 中的 operation 不受此影响
>   - 注意：<font color='red'>operation 的队列优先级只决定当前所有 isReady 状态为 YES 的 operation 的执行顺序。</font>
>
> - **@property double ~~threadPriority~~**：修改 **Operation** 执行任务线程的优先级（iOS 8后已废弃，替换为**qualityOfService**）
>
> - - 从 iOS 4.0 开始，我们可以修改 operation 的执行任务线程的优先级
>   - 可以给 operation 的线程优先级指定一个从 0.0 到 1.0 的浮点数值，0.0 表示最低的优先级，1.0 表示最高的优先级，默认值为 0.5 。
>   - 注意：只能在执行一个 operation 或将其添加到 operation queue 前，通过 operation 的 setThreadPriority: 方法来修改它的线程优先级。当 operation 开始执行时，NSOperation 类中默认的 start 方法会使用我们指定的值来修改当前线程的优先级
>   - 注意：指定的这个线程优先级只会影响 main 方法执行时所在线程的优先级。所有其它的代码，包括 operation 的 completion block 所在的线程会一直以默认的线程优先级执行。
>
> - 手动执行 **Operation**：调用 **-(void)start** 方法
>
> - - 会根据 isAsynchronous 属性（<font color='orange' >默认是NO，所以以同步任务方式执行</font>），~~动态判断是否阻塞当前线程~~ 应该是此任务是否同步执行
>
> - 取消操作
>
> - - \- (void)cancel：可取消操作，实质是标记 isCancelled 状态
>
> - 判断操作状态
>
> - - \- (BOOL)isFinished：标记操作是否已经结束
>   - \- (BOOL)isCancelled：标记操作是否已经取消
>   - \- (BOOL)isExecuting：标记操作是否正在执行
>   - \- (BOOL)isReady：标记操作是否已经就绪，这个值和操作的依赖关系相关。

## 2.1 NSInvocationOperation

NSInvocationOperation 是 NSOperation 类的一个子类，当一个 NSInvocationOperation 开始执行时，它会调用我们指定的 object 的 selector 方法。

- - 通过使用 NSInvocationOperation 类，我们可以避免为每一个任务都创建一个自定义的子类，特别是当我们在修改一个已经存在的应用，并且这个应用中已经有了我们需要执行的任务所对应的 object 和 selector 时非常有用

- - 在没有使用 NSOperationQueue ，只单独使用子类 [NSInvocationOperation start] 执行一个操作的情况下，操作是在当前线程执行的，不会开启新线程

## 2.2 NSBlockOperation

我们可以使用 NSBlockOperation 来并发执行一个或多个 block      ，只有当一个 NSBlockOperation 所关联的所有 block 都执行完毕时，这个 NSBlockOperation 才算执行完成，有点类似于 `dispatch_group` 的概念。

NSBlockOperation 是 NSOperation 类的另外一个系统预定义的子类，我们可以用它来封装一个或多个 `block` 。我们知道 `GCD` 主要就是用来进行 `block` 调度的，那为什么我们还需要      NSBlockOperation 类呢？一般来说，有以下两个场景我们会优先使用 NSBlockOperation 类：

- 当我们在应用中已经使用了 Operation Queues 且不想创建 Dispatch Queues 时，NSBlockOperation        类可以为我们的应用提供一个面向对象的封装；
- <font color='red'>我们需要用到 Dispatch Queues 不具备的功能时，比如需要设置 operation 之间的依赖关系、使用 `KVO` 观察operation 的状态变化等。</font>
- NSBlockOperation 的使用：

> * `+ (instancetype)blockOperationWithBlock:(void (^)(void))block：`:  **初始化Operation**
>
> ​        在没有使用 NSOperationQueue ，只单独使用子类 [NSBlockOperation start] 执行一个操作的情况下，操作是在当前线程执行的，不会开启新线程（~~网上很多互相抄袭的博客中说，该 block 可能不在当前线程执行？目前没有验证出，待深入研究。。。。~~）
>
> 👆🏻<font color='red'>划线处已验证</font>：<font color='orange'>blockOperation中只添加了一个执行任务的话是在当前线程执行；添加了多个任务的话是并发执行; 另外，首先执行的任务会在当前线程，其他任务在其他线程；start后的代码，是在所有任务执行完毕才调用！</font>
>
> * `- (void)addExecutionBlock:(void (^)(void))block;`:  **向一个Operation实例中追加任务**
>
> 实例代码：
>
> ```objective-c
>     /// NSBlockOperation基本使用
> - (void)testOperation_block_usage {
>     NSLog(@"🌈 %@ begin: %@", NSStringFromSelector(_cmd),[NSThread currentThread]);
>     NSBlockOperation *op =  [NSBlockOperation blockOperationWithBlock:^{
>         NSLog(@"1/3 %@ Start, mainThread(33): %@, currentThread(%ld): %@", NSStringFromSelector(_cmd), [NSThread mainThread], [NSThread currentThread].qualityOfService,[NSThread currentThread]);
>         sleep(10);
>         NSLog(@"1/3 Finish executing .");
>     }];
>     [op addExecutionBlock:^{
>         NSLog(@"2/3 %@ Start, mainThread(33): %@, currentThread(%ld): %@", NSStringFromSelector(_cmd), [NSThread mainThread], [NSThread currentThread].qualityOfService,[NSThread currentThread]);
>         sleep(3);
>         NSLog(@"2/3 Finish executing .");
>     }];
>     [op addExecutionBlock:^{
>         NSLog(@"3/3 %@ Start, mainThread(33): %@, currentThread(%ld): %@", NSStringFromSelector(_cmd), [NSThread mainThread], [NSThread currentThread].qualityOfService,[NSThread currentThread]);
>         sleep(30);
>         NSLog(@"3/3 Finish executing .");
>     }];
>      
> 
>     // 回调监听（completionBlock是异步的，使用任务都执行完才会回调）
>     op.completionBlock = ^{
>         NSLog(@" operation完成了: %@", [NSThread currentThread]);
>     };
>     // 3个block是并行执行的，所以他们的完成顺序不确定
>   // 因为 NSBlockOperation 的 isAsynchronous 默认是NO，所以会阻塞 start 方法后的任务，直到所有 block 都完成，🌈 over才会打印
>     [op start];
>     NSLog(@"🌈 %@ over: %@", NSStringFromSelector(_cmd),[NSThread currentThread]);
> }
> 
> ```
>
> 

# 3. 自定义 Operation

`NSOperation` 本身是个抽象类，在使用前必须子类化(系统预定义了两个子类：`NSInvocationOperation` 和 `NSBlockOperation`)。那问题来了，在子类化过程中，需要重写父类的哪些方法？

首先就要了解一下`NSOperation`类中几个重要方法的默认实现：

| 方法         | 实现                                                         |
| ------------ | ------------------------------------------------------------ |
| main         | 空                                                           |
| start        | 1. 随Operation执行过程修改其状态（非常重要）<br />2. 调用main方法<br />3. 错误检查，如：operation是否完成或cancel（此时不会调用main方法）、正在执行或尚未准备就绪（抛出异常NSInvalidArgumentException） |
| cancel       | 修改operation的内部状态标识位（将isCanceled属性置为YES，仅此而已） |
| ready        | 根据依赖关系确定是否准备就绪（当该Operation没有依赖的operation时返回YES） |
| asynchronous | 返回NO                                                       |

在 NSOperation 中还有一个重要概念：operation 的状态，并且当状态变化时需要通过 KVO 的方式通知外部：

| 监听状态（KeyPath） | 描述                                                         |
| ------------------- | ------------------------------------------------------------ |
| isReady             | 表示operation是否准备就绪可以执行；<br />默认isReady的值取决于是否有依赖的operations没有执行完。<br />（ps:当cancel某个正在因依赖关系而等待的operation时，系统将忽略依赖关系，直接将该operation的isReady属性置为YES，其目的是让operation queue能尽快清理该operation。） |
| isExecuting         | 表示operation是否正在执行；<br />如果重写start方法需要正确设置isExecuting的值并抛出KVO通知。<br /> |
| isFinished          | 表示operation是否已完成（cancel被认为是完成）；<br />如果重写start方法，在operation完成时需要将该属性置为YES并抛出KVO通知。<br /><br />将isFinished置为YES，并抛出KVO通知至少有以下3个重要作用：<br /><br />1. 清除其他operation对该operation的依赖（如果有）；<br />2. operation queue将该operation出队;<br />3. 执行completionBlock（如果有）。 |
| isCancelled         | 表示operation是否被取消，一般由实例方法`cancel`负责，不需要手动操作该属性。 |

回到前面那个问题：子类化 NSOperation 时需要重写哪些方法？
这取决于子类化后的 operation 是 Synchronous 还是 Asynchronous(NSOperation 默认是Synchronous)。

## 3.1 **Synchronous VS. Asynchronous Operations**

<font color='red'>由于操作 NSOperation 与 NSOperation 任务的执行往往在不同的线程上进行，在继续之前需要强调线程安全问题：『NSOperation 本身是 thread-safe，*当我们在子类重写或自定义方法时同样需要保证 thread-safe*』。</font>

### Synchronous Operations

对于 Synchronous Operation，在调用其 `start` 方法的线程上同步执行该 operation 的任务，`start` 方法返回时 operation 执行完成。因此，对于 Synchronous Operation 一般只需重写 `main` 方法即可(`start`方法的默认实现已实现相关 KVO 功能)。

### Asynchronous Operations

然而对于 Asynchronous Operation，调用其 `start` 方法后，在 `start` 返回时 operation 的任务可能还没完成(为了实现异步，一般需要在其他线程执行 operation 的具体任务)。因此 `start` 方法默认实现不能满足异步需要(默认实现会在`start`返回前将 `isExecuting` 置为 NO、`isFinished` 置为 YES，并产生 KVO 通知)。此时至少需要重写以下方法：

- start：
  我们知道 NSOperation 本身不具备并发(或者说异步执行)能力，因此需要 `start` 方法来实现，可以通过创建子线程或其他异步方式完成。同时需要在任务开始前将 `isExecuting` 置为YES 并抛出 KVO 通知。
  ***『重写的 `start` 方法一定不能调用 `[super start]`』\***
- asynchronous
  返回 YES，一般不需要抛出 KVO 通知
- executing
  返回 operation 的执行状态，在其值发生变化时需要在 `isExecuting` 上抛出 KVO 通知
- finished
  返回 operation 的完成状态，同样值变化时需要在 `isFinished` 上抛出 KVO 通知

 这里我们看看著名的网络框架 AFNetworking 中关于 NSOperation 的使用：

> AFNetworking 3.0 全面使用 `NSURLSession`，而 `NSURLSession` 本身是异步的、且没有 `NSURLConnection` 需要 runloop 配合的问题，因此在3.0版本中并没有使用 NSOperation，代码得到很大的简化。这里我们说的是 AFNetworking 2.3.1 版本。

在 AFNetworking 中 AFURLConnectionOperation 是个异步的 NSOperation 子类，其 `start` 方法如下：

![img](http://zxfcumtcs.github.io/img/AFURLConnectionOperation-start.png)

从上面 `start` 方法的实现可以看到：

1. 用 lock(递归锁) 保证了thread-safe；
2. 检查了 operation 是否已被 cancel；
3. 检查了 operation 是否已 ready；
4. 通过子线程实现并发；
5. 在 state setter 中实现了 KVO。

![setter state](http://zxfcumtcs.github.io/img/AFURLConnectionOperation-statesetter.png)

再来看看 AFURLConnectionOperation 使用的子线程：

![](http://zxfcumtcs.github.io/img/AFURLConnectionOperation-thread.png)

可以看到，所有 AFURLConnectionOperation 实例底层使用的是同一个子线程，并在该线程中启动了 runloop（`NSURLConnection` 的网络回调必须要有 runloop 的配合，通过port-based input source 唤醒 runloop 处理网络事件），也就是说 AFURLConnectionOperation 是在一条常驻子线程中处理网络回调。

前面我们提到 operation 被 cancel 时也被认为是完成，这点在自定义 `start` 时同样需要注意：

![](http://zxfcumtcs.github.io/img/AFURLConnectionOperation-cancel.png)

在 AFURLConnectionOperation 的 `cancelConnection` 以及 `connection:didFailWithError:` 方法中都会调用其 `finish` 方法：

![](http://zxfcumtcs.github.io/img/AFURLConnectionOperation-finish.png)

ps：虽然 NSOperation 支持 cancel，但在调用 `cancel` 方法后该如何处理完全由我们自定义的 `start` 方法决定(当然良好的设计应该要符合 cancel 的语义)。

同时，AFURLConnectionOperation 也实现了以下方法：

![](http://zxfcumtcs.github.io/img/AFURLConnectionOperation-isReady-isFinished.png)



# **4. 关于 NSOperation 其他细节问题**

- dependencies:
  我们可以在 operation 间添加依赖关系，在某个 operation 所依赖的 operations 完成之前，其一直处于未就绪状态(`isReady` 为 NO)。
  需要注意的是，依赖关系是 operation 自身的状态，也就是说有依赖关系的 operations 可以处在不同的 NSOperationQueue 中。
- isReady:
  `isReady` 默认实现主要处理 operation 间的依赖关系，当我们自定义该方法时需要考虑 `super` 的值，如 AFURLConnectionOperation中关于 `isReady` 的实现：[![img](http://zxfcumtcs.github.io/img/AFURLConnectionOperation-isReady.png)](http://zxfcumtcs.github.io/img/AFURLConnectionOperation-isReady.png)
- qualityOfService:
  ~~用于表示 operation 在获取系统资源时的优先级，默认值：`NSQualityOfServiceBackground`~~，(❌，<font color="orange">苹果注释中是background，实际代码验证默认是default</font>)，我们可以根据需要给 operation 赋不同的优化级，如最高优化级：`NSQualityOfServiceUserInteractive`。
- queuePriority:
  用于设置 operation 在 operation queue 中的相对优化级，同一 queue 中优化级高的 operation(`isReady` 为 YES) 会被优先执行。需要注意区分`qualityOfService`(在系统层面，operation 与其他线程获取资源的优先级)与`queuePriority`(同一 queue 中 operation 间执行的优化级)的区别。
  同时，需要注意`dependencies`(严格控制执行顺序)与`queuePriority`(queue 内部相对优先级)的区别。