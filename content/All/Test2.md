# 1、Crash

> 引自：https://www.jianshu.com/p/3f6775c02257

## 1.1 常见的Crash类型有哪些？

>根据Crash 的不同来源，Crash 分为以下三类：
>
>- Mach 异常
>
>  最底层的内核级异常。用户态的开发者可以直接通过Mach API设置thread，task，host的异常端口，来捕获Mach异常。
>
>- Unix 信号
>
>  又称BSD 信号，如果开发者没有捕获Mach异常，则会被host层的方法ux_exception()将异常转换为对应的UNIX信号，并通过方法threadsignal()将信号投递到出错线程。可以通过方法signal(x, SignalHandler)来捕获signal。
>
>- NSException
>
>  应用级异常，它是未被捕获的Objective-C异常，导致程序向自身发送了SIGABRT信号而崩溃，是app自己可控的，对于未捕获的Objective-C异常，是可以通过try catch来捕获的，或者通过NSSetUncaughtExceptionHandler()机制来捕获。

## 1.2 分别用什么方式可以捕获？

 ### 1. 如何捕获Mach异常？

><img src="/Users/tangh/yuki/博客/文章仓库/YLNoteHub/content/All/image/crash-mach_1.jpeg" alt="img" style="zoom:30%;" />
>
>参考上图，主要的流程是：新建一个监控线程，在监控线程中监听 Mach 异常并处理异常信息。主要的步奏如下图：
>
><img src="/Users/tangh/yuki/博客/文章仓库/YLNoteHub/content/All/image/crash-mach_2.png" alt="内核 crash 手机的流程" style="zoom:120%;" />
>
>具体代码如下:
>
>```c++
>static mach_port_t server_port;
>static void *exc_handler(void *ignored);
>
>//判断是否 Xcode 联调
>bool ksdebug_isBeingTraced(void)
>{
>struct kinfo_proc procInfo;
>size_t structSize = sizeof(procInfo);
>int mib[] = {CTL_KERN, KERN_PROC, KERN_PROC_PID, getpid()};
>
>if(sysctl(mib, sizeof(mib)/sizeof(*mib), &procInfo, &structSize, NULL, 0) != 0)
>{
>    return false;
>}
>
>return (procInfo.kp_proc.p_flag & P_TRACED) != 0;
>}
>
>#define EXC_UNIX_BAD_SYSCALL 0x10000 /* SIGSYS */
>#define EXC_UNIX_BAD_PIPE    0x10001 /* SIGPIPE */
>#define EXC_UNIX_ABORT       0x10002 /* SIGABRT */
>static int signalForMachException(exception_type_t exception, mach_exception_code_t code)
>{
>switch(exception)
>{
>    case EXC_ARITHMETIC:
>        return SIGFPE;
>    case EXC_BAD_ACCESS:
>        return code == KERN_INVALID_ADDRESS ? SIGSEGV : SIGBUS;
>    case EXC_BAD_INSTRUCTION:
>        return SIGILL;
>    case EXC_BREAKPOINT:
>        return SIGTRAP;
>    case EXC_EMULATION:
>        return SIGEMT;
>    case EXC_SOFTWARE:
>    {
>        switch (code)
>        {
>            case EXC_UNIX_BAD_SYSCALL:
>                return SIGSYS;
>            case EXC_UNIX_BAD_PIPE:
>                return SIGPIPE;
>            case EXC_UNIX_ABORT:
>                return SIGABRT;
>            case EXC_SOFT_SIGNAL:
>                return SIGKILL;
>        }
>        break;
>    }
>}
>return 0;
>}
>
>static NSString *stringForMachException(exception_type_t exception) {
>switch(exception)
>{
>    case EXC_ARITHMETIC:
>        return @"EXC_ARITHMETIC";
>    case EXC_BAD_ACCESS:
>        return @"EXC_BAD_ACCESS";
>    case EXC_BAD_INSTRUCTION:
>        return @"EXC_BAD_INSTRUCTION";
>    case EXC_BREAKPOINT:
>        return @"EXC_BREAKPOINT";
>    case EXC_EMULATION:
>        return @"EXC_EMULATION";
>    case EXC_SOFTWARE:
>    {
>        return @"EXC_SOFTWARE";
>        break;
>    }
>}
>return 0;
>}
>
>void installExceptionHandler() {
>if (ksdebug_isBeingTraced()) {
>    // 当前正在调试状态, 不启动 mach 监听
>    return ;
>}
>kern_return_t kr = mach_port_allocate(mach_task_self(), MACH_PORT_RIGHT_RECEIVE, &server_port);
>assert(kr == KERN_SUCCESS);
>
>kern_return_t rc = 0;
>exception_mask_t excMask = EXC_MASK_BAD_ACCESS |
>EXC_MASK_BAD_INSTRUCTION |
>EXC_MASK_ARITHMETIC |
>EXC_MASK_SOFTWARE |
>EXC_MASK_BREAKPOINT;
>
>rc = mach_port_allocate(mach_task_self(), MACH_PORT_RIGHT_RECEIVE, &server_port);
>if (rc != KERN_SUCCESS) {
>    fprintf(stderr, "------->Fail to allocate exception port\\\\\\\\n");
>    return;
>}
>
>rc = mach_port_insert_right(mach_task_self(), server_port, server_port, MACH_MSG_TYPE_MAKE_SEND);
>if (rc != KERN_SUCCESS) {
>    fprintf(stderr, "-------->Fail to insert right");
>    return;
>}
>
>rc = thread_set_exception_ports(mach_thread_self(), excMask, server_port, EXCEPTION_DEFAULT, MACHINE_THREAD_STATE);
>if (rc != KERN_SUCCESS) {
>    fprintf(stderr, "-------->Fail to  set exception\\\\\\\\n");
>    return;
>}
>
>//建立监听线程
>pthread_t thread;
>pthread_create(&thread, NULL, exc_handler, NULL);
>}
>
>static void *exc_handler(void *ignored) {
>// Exception handler – runs a message loop. Refactored into a standalone function
>// so as to allow easy insertion into a thread (can be in same program or different)
>mach_msg_return_t rc;
>fprintf(stderr, "Exc handler listening\\\\\\\\n");
>// The exception message, straight from mach/exc.defs (following MIG processing) // copied here for ease of reference.
>typedef struct {
>    mach_msg_header_t Head;
>    /* start of the kernel processed data */
>    mach_msg_body_t msgh_body;
>    mach_msg_port_descriptor_t thread;
>    mach_msg_port_descriptor_t task;
>    /* end of the kernel processed data */
>    NDR_record_t NDR;
>    exception_type_t exception;
>    mach_msg_type_number_t codeCnt;
>    integer_t code[2];
>    int flavor;
>    mach_msg_type_number_t old_stateCnt;
>    natural_t old_state[144];
>} Request;
>
>Request exc;
>
>struct rep_msg {
>    mach_msg_header_t Head;
>    NDR_record_t NDR;
>    kern_return_t RetCode;
>} rep_msg;
>
>for(;;) {
>    // Message Loop: Block indefinitely until we get a message, which has to be
>    // 这里会阻塞，直到接收到exception message，或者线程被中断。
>    // an exception message (nothing else arrives on an exception port)
>    rc = mach_msg( &exc.Head,
>                  MACH_RCV_MSG|MACH_RCV_LARGE,
>                  0,
>                  sizeof(Request),
>                  server_port, // Remember this was global – that's why.
>                  MACH_MSG_TIMEOUT_NONE,
>                  MACH_PORT_NULL);
>
>        if(rc != MACH_MSG_SUCCESS) {
>        /*... */
>        break ;
>    };
>
>        //Mach Exception 类型
>    NSMutableString *crashInfo = [NSMutableString stringWithFormat:@"mach exception:%@ %@\n\n",stringForMachException(exc.exception), stringForSignal(signalForMachException(exc.exception, exc.code[0]))];
>
>        rep_msg.Head = exc.Head;
>    rep_msg.NDR = exc.NDR;
>    rep_msg.RetCode = KERN_FAILURE;
>
>        kern_return_t result;
>    if (rc == MACH_MSG_SUCCESS) {
>        result = mach_msg(&rep_msg.Head,
>                          MACH_SEND_MSG,
>                          sizeof (rep_msg),
>                          0,
>                          MACH_PORT_NULL,
>                          MACH_MSG_TIMEOUT_NONE,
>                          MACH_PORT_NULL);
>    }
>    //移除其他 Crash 监听, 防止死锁
>    NSSetUncaughtExceptionHandler(NULL);
>    signal(SIGHUP, SIG_DFL);
>    signal(SIGINT, SIG_DFL);
>    signal(SIGQUIT, SIG_DFL);
>    signal(SIGABRT, SIG_DFL);
>    signal(SIGILL, SIG_DFL);
>    signal(SIGSEGV, SIG_DFL);
>    signal(SIGFPE, SIG_DFL);
>    signal(SIGBUS, SIG_DFL);
>    signal(SIGPIPE, SIG_DFL);
>}
>
>return  NULL;
>}
>```
>
>监听 Mach 异常需要注意:
>
>- 避免在 Xcode 联调时监听
>
> 原因是我们监听了`EXC_BREAKPOINT`这类型的`Exception`，一旦启动 app 联调后， 会立即触发`EXC_BREAKPOINT`。而这段代码处理完后，会进入下一个循环等待，可主线程这是还等着消息处理结果，这就造成等待死锁。

### 2. 如何捕捉 Unix 信号？

> 一般来说我们需要捕捉以下信号:
>
> ```c
> static const int g_fatalSignals[] =
> {
>  SIGABRT,
>  SIGBUS,
>  SIGFPE,
>  SIGILL,
>  SIGPIPE,
>  SIGSEGV,
>  SIGSYS,
>  SIGTRAP,
> };
> ```
>
> 而要捕捉 Unix 信号，比 Mach 异常容易多了
>
> ```objective-c
> void installSignalHandler() {
>      signal(SIGABRT, handleSignalException);
>  //...等等其他需要监听的 Signal
> }
> void handleSignalException(int signal) {
>  //打印堆栈
>  NSMutableString * crashInfo = [[NSMutableString alloc]init];
>  [crashInfo appendString:[NSString stringWithFormat:@"signal:%d\n",signal]];
>  [crashInfo appendString:@"Stack:\n"];
>  void* callstack[128];
>  int i, frames = backtrace(callstack, 128);
>  char** strs = backtrace_symbols(callstack, frames);
>  for (i = 0; i <frames; ++i) {
>      [crashInfo appendFormat:@"%s\n", strs[I]];
>  }
>  NSLog(@"%@", crashInfo);
>  //移除其他 Crash 监听, 防止死锁
>  NSSetUncaughtExceptionHandler(NULL);
>  signal(SIGHUP, SIG_DFL);
>  signal(SIGINT, SIG_DFL);
>  signal(SIGQUIT, SIG_DFL);
>  signal(SIGABRT, SIG_DFL);
>  signal(SIGILL, SIG_DFL);
>  signal(SIGSEGV, SIG_DFL);
>  signal(SIGFPE, SIG_DFL);
>  signal(SIGBUS, SIG_DFL);
>  signal(SIGPIPE, SIG_DFL);
> }
> ```
>
> ## 备用信号栈
>
> 上面这个方法可以监控到大部分的 Signal 异常，但是我们会发现如果遇到死循环这类的Crash，就没法监控了。原因是一般情况下，信号处理函数被调用时，内核会在进程的栈上为其创建一个栈帧。但这里就会有一个问题，如果之前栈的增长**达到了栈的最大长度**，或是栈没有达到最大长度但也比较接近，那么就会导致信号处理函数不能得到足够栈帧分配。
>
> 为了解决这个问题，我们需要设定一个**可选的栈帧**：
>
> 1. 申请一块内存空间作为可选的信号处理函数栈使用
> 2. 使用 sigaltstack 函数通知系统可选的信号处理栈帧的存在及其位置
> 3. 当使用 sigaction 函数建立一个信号处理函数时，通过指定 SA_ONSTACK 标志通知系统这个信号处理函数应该在可选的栈帧上面执行注册的信号处理函数
>
> 前面监听 Unix 信号的代码，改动一下：
>
> 
>
> ```c
> void installSignalHandler() {
>         stack_t ss;
>     struct sigaction sa;
>     struct timespec req, rem;
>     long ret;
> 
>     ss.ss_flags = 0;
>     ss.ss_size = SIGSTKSZ;
>     ss.ss_sp = malloc(ss.ss_size);
>     sigaltstack(&ss, NULL);
> 
>     memset(&sa, 0, sizeof(sa));
>     sa.sa_handler = handleSignalException;
>     sa.sa_flags = SA_ONSTACK;
>     sigaction(SIGABRT, &sa, NULL);
> }
> ```

### 3. 如何捕获NSException？

>NSException 是应用级异常，是指 OC 代码运行过程由Objective-C 抛出的异常，基本上是代码运行过程中的逻辑错误。比如往 NSArray 中插入 nil 对象，或者用nil 初始化 NSURL 等。最简单区分一个异常是否 NSException 的方式是看这个异常能否被@trycatch 给捕获。
>
>## 常见的 NSException 场景
>
>- 非主线程刷新UI
>- NSInvalidArgumentException
>  非法参数异常(NSInvalidArgumentException)是 Objective – C 代码最常出现的错误，所以平时在写代码的时候，需要多加注意，加强对参数的检查，避免传入非法参数导致异常，其中尤以nil参数为甚。
>- NSRangeException
>  越界异常(NSRangeException)也是比较常出现的异常。
>- NSGenericException
>  NSGenericException这个异常最容易出现在foreach操作中，在for in循环中如果修改所遍历的数组，无论你是add或remove，都会出错 “for in”,它的内部遍历使用了类似 Iterator进行迭代遍历，一旦元素变动，之前的元素全部被失效，所以在foreach的循环当中，最好不要去进行元素的修改动作，若需要修改，循环改为for遍历，由于内部机制不同，不会产生修改后结果失效的问题。
>- NSInternalInconsistencyException
>  不一致导致出现的异常
>   比如NSDictionary当做NSMutableDictionary来使用，从他们内部的机理来说，就会产生一些错误
>   NSMutableDictionary *info = method return to NSDictionary type;
>   [info setObject:@“sxm” forKey:@”name”];
>   比如xib界面使用或者约束设置不当
>- NSFileHandleOperationException
>  处理文件时的一些异常，最常见的还是存储空间不足的问题，比如应用频繁的保存文档，缓存资料或者处理比较大的数据:
>   所以在文件处理里，需要考虑到手机存储空间的问题。
>- NSMallocException
>  这也是内存不足的问题，无法分配足够的内存空间
>   此外还有
>- KVO Crash
>  移除未注册的观察者
>   重复移除观察者
>   添加了观察者但是没有实现`-observeValueForKeyPath:ofObject:change:context:`方法
>   添加移除keypath=nil
>   添加移除observer=nil
>- unrecognized selector send to instance
>
>## 监听 NSException 异常
>
>NSException的监听也十分简单:
>
>```c
>void InstallUncaughtExceptionHandler(void) {
>   NSSetUncaughtExceptionHandler( &handleUncaughtException );
>}
>
>void handleUncaughtException(NSException *exception) {
>   NSString * crashInfo = [NSString stringWithFormat:@"yyyy Exception name：%@\nException reason：%@\nException stack：%@",[exception name], [exception reason], [exception callStackSymbols]];
>   NSLog(@"%@", crashInfo);
>}
>```
>
>需要注意的是，在监听处理的方法中，是**无法直接采集错误到堆栈**的。详情我同样会在下一篇的崩溃堆栈收集的文章中介绍。

## 1.3 具体问题

### 1. 怎么降低应用的线上crash率？

综合，基本都是通过hook原始方法替换成安全方法解决。具体情况根据方法处理内容决定。

> ### UIKit Called on Non-Main Thread
>
> UIKit不是线程安全的，执行UIKit操作如果不在主线程很可能造成程序Crash。所以对UIView 的setNeedsLayout，layoutIfNeeded，layoutSubviews，setNeedsUpdateConstraints方法进行hook。如果执行以上函数没有在主队列，dispatch到主线程执行。
>
> ```objective-c
> - (void)wt_safe_setNeedsLayout
> {
>     if(![NSThread isMainThread]){
>         dispatch_async(dispatch_get_main_queue(), ^{
>             NSAssert(false, @"wt_safe_setNeedsLayout failed");
>             [self wt_safe_setNeedsLayout];
>         });
>     }else{
>         [self wt_safe_setNeedsLayout];
>     }
> }
> ```
>
> ### 避免 Foundation 类Carsh
>
> ### NSString
>
> ```objectivec
> + (instancetype)stringWithUTF8String:(const char *)bytes
> - (instancetype)initWithString:(NSString *)aString
> - (instancetype)initWithUTF8String:(const char *)nullTerminatedCString
> - (instancetype)initWithFormat:(NSString *)format locale:(id)locale arguments:(va_list)argList
> - (NSString *)stringByAppendingString:(NSString *)aString
> - (unichar)characterAtIndex:(NSUInteger)index
> - (void)getCharacters:(unichar *)buffer range:(NSRange)range
> - (NSRange)rangeOfCharacterFromSet:(NSCharacterSet  *)searchSet 
>                             options:(NSStringCompareOptions)mask
>                               range:(NSRange)searchRange
>                                     
> - (NSRange)rangeOfString:(NSString *)searchString
>                 options:(NSStringCompareOptions)mask
>                   range:(NSRange)searchRange
>                  locale:(NSLocale *)locale
> - (NSString *)substringFromIndex:(NSUInteger)from
> - (NSString *)substringWithRange:(NSRange)range
> - (NSString *)substringToIndex:(NSUInteger)to
> - (void)getLineStart:(NSUInteger *)startPtr   
>                  end:(NSUInteger *)lineEndPtr                                         
>                  contentsEnd:(NSUInteger *)contentsEndPtr
>                    forRange:(NSRange)range
> ```
>
> ### NSAttributedString
>
> hook 方法：对传入参数range 进行check，如果range有问题，直接返回nil
>
> ```objectivec
> - (NSAttributedString *)attributedSubstringFromRange:(NSRange)range;
> ```
>
> ### NSFileManager
>
> ```objectivec
> - (nullable NSDirectoryEnumerator<NSURL *> *)enumeratorAtURL:(NSURL *)url includingPropertiesForKeys:(nullable NSArray<NSURLResourceKey> *)keys options:(NSDirectoryEnumerationOptions)mask errorHandler:(nullable BOOL (^)(NSURL *url, NSError *error))handler
> ```
>
> ### NSIndexPath
>
> ```objectivec
> - (void)getIndexes:(NSUInteger *)indexes range:(NSRange)positionRang
> ```
>
> ### NSJSONSerialization
>
> ```objectivec
> + (NSData *)dataWithJSONObject:(id)obj options:(NSJSONWritingOptions)opt error:(NSError **)error
> ```
>
> ### NSDictionary
>
> hook 方法：
>
> ```objectivec
> + (id)sharedKeySetForKeys:(NSArray<KeyType <NSCopying>> *)keys
> - (instancetype)initWithObjects:(const ObjectType _Nonnull [_Nullable])objects forKeys:(const KeyType <NSCopying> _Nonnull [_Nullable])keys
> - (instancetype)initWithObjects:(const ObjectType _Nonnull [_Nullable])objects forKeys:(const KeyType <NSCopying> _Nonnull [_Nullable])keys count:(NSUInteger)cnt
> ```
>
> ### NSMutableDictionary
>
> hook 方法：
>
> ```erlang
> + (NSMutableDictionary<KeyType, ObjectType> *)dictionaryWithSharedKeySet:(id)keyset
> - (void)setObject:(ObjectType)anObject forKey:(KeyType <NSCopying>)aKey;
> - (void)removeObjectForKey:(KeyType)aKey;
> ```
>
> ### NSSet
>
> ```erlang
> - (instancetype)WT_initWithObjects:(const id [])objects count:(NSUInteger)cnt
> - (void)addObject:(id)object;
> - (void)makeObjectsPerformSelector:(SEL)aSelector
> - (void)makeObjectsPerformSelector:(SEL)aSelector
>                         withObject:(id)argument
> ```
>
> ### NSMutableSet
>
> ```erlang
> - (void)addObject:(id)anObject
> ```
>
> ### NSMutableString
>
> ```objectivec
> - (void)setString:(NSString *)aString
> - (void)appendString:(NSString *)aString
> - (void)deleteCharactersInRange:(NSRange)range
> - (void)insertString:(NSString *)aString atIndex:(NSUInteger)loc
> - (void)replaceCharactersInRange:(NSRange)range withString:(NSString *)aString
> -  (NSUInteger)replaceOccurrencesOfString:(NSString *)target
>                                      withString:(NSString *)replacement
>                                         options:(NSStringCompareOptions)options
>                                           range:(NSRange)searchRange
> ```
>
> \###NSURL
>
> ```objectivec
> + (NSURL *)fileURLWithPath:(NSString *)path
> + (NSURL *)fileURLWithPath:(NSString *)path isDirectory:(BOOL)isDir
> + (NSURL *)fileURLWithPathComponents:(NSArray<NSString *> *)components
> + (NSURL *)fileURLWithPath:(NSString *)path
>                       isDirectory:(BOOL)isDir
>                     relativeToURL:(NSURL *)baseURL
> - (instancetype)initWithString:(NSString *)URLString relativeToURL:(NSURL *)baseURL
> - (instancetype)initFileURLWithPath:(NSString *)path
> - (instancetype)initFileURLWithPath:(NSString *)path
>                              relativeToURL:(NSURL *)baseURL
> - (instancetype)initFileURLWithPath:(NSString *)path
>                                isDirectory:(BOOL)isDir
>                              relativeToURL:(NSURL *)baseURL
> ```
>
> 1. KVO
> 2. 容器越界（NSArray， NSDictionary,...）
> 3. unrecognized selector crash (这个很多时候是由于class使用错误导致)
> 4. NSTimer 导致crash
>
> ### KVO Crash
>
> 项目中KVO crash 占比很高， 主要原因为，添加删除不对称导致。 解决方法为，添加Map进行缓存。
>
> ### unrecognized selector crash
>
> ```objective-c
>     [NSObject jr_swizzleMethod:@selector(forwardingTargetForSelector:) withMethod:@selector(WT_safeForwardingTargetForSelector:) error:&error];
>     
>     - (id)WT_safeForwardingTargetForSelector:(SEL)aSelector
> {
>     NSMethodSignature *signature = [self methodSignatureForSelector:aSelector];
>     if ([self respondsToSelector:aSelector] || signature) {
>         return [self WT_safeForwardingTargetForSelector:aSelector];
>     }
>     
>     return [WTSafeGuard createFakeForwardTargetObject:self selector:aSelector];
> }
> ```
>
> 

### 2. 收到一个线上crash，你一般怎么进行解析？

### 3. 通过线上防护可以降低crash率？怎么进行全局的crash线上防护？

### 4. 像野指针这种crash怎么进行线上防护？

# 2、私有Api

怎么调用私有Api？有什么问题吗？审核不过有什么办法？

## 2.1 怎么调用私有Api

* 直接调用法

  因为私有 API 没有暴露出来，编译会报错。可以添加匿名 `Category` 声明下私有方法。

  ```objc
  @interface UIView()
  -(id)recursiveDescription;
  @end
  ```

* 字符拼接

  借助 Objective-C 语言的动态特性，在运行时用 `performSelector` 执行拼接的 selector 方法：

* 代码混淆

  ```objc
  // statusBar
  NSData *data = [NSData dataWithBytes:(unsigned char[]){0x73, 0x74, 0x61, 0x74, 0x75, 0x73, 0x42, 0x61, 0x72} length:9]; 
  NSString *key = [[NSString alloc] initWithData:data encoding:NSASCIIStringEncoding];
  ```

## 2.2 存在问题

可能无法过审

## 2.3 检测方法

###  方法1：符号表检查

用 `nm`、`otool` 等工具导出二进制包的函数符号表，以检查私有 API 的调用。

开源项目 [iOS-private-api-checker](https://github.com/NetEaseGame/iOS-private-api-checker) 以这种方式实现了对私有 API 调用的检查。

然而这种方法的缺点是，无法检测字符串拼接方法的私有 API 调用。

###  方法2：运行时分析

在审核人员运行 App 的同时，用 runtime 工具检测是否调用了私有 API。具体原理待补充。

这种方法的漏洞，当审核人员无法进入调用了私有 API 的功能时（通过后台下发配置文件控制功能入口），会有漏测的情况。

### 方法3：静态[代码分析](https://cloud.tencent.com/product/tcap?from=10680)

为检测字符串拼接法调用私有 API，受论文 [1] 启发，可以在对二进制文件反汇编结果的基础上，进行静态分析：

1. 找出动态调用 API 方法如 `performSelector:` ，以及调用对象的类
2. 检查参数，如果参数是拼接方法生成，推导求得拼接的结果
3. 根据 1 2 判断是否调用了私有 API

以私有 API 调用方法2 的代码为例，用 Hopper 对其反汇编，得到伪代码：

```js
void -[ViewController viewDidLoad](void * self, void * _cmd) {
    r31 = r31 - 0x60;
    *(r31 + 0x30) = r22;
    *(0x40 + r31) = r21;
    *(r31 + 0x40) = r20;
    *(0x50 + r31) = r19;
    *(r31 + 0x50) = r29;
    *(0x60 + r31) = r30;
    *(r31 + 0x28) = ___stack_chk_guard;
    *(r31 + 0x8) = self;
    *(r31 + 0x10) = 0x100008d38;
    [[r31 + 0x8 super] viewDidLoad];
    *(r31 + 0x18) = @"_private";
    *(0x28 + r31) = @"Method";
    r0 = [0x100008d28 arrayWithObjects:r31 + 0x18 count:0x2];
    r0 = [r0 retain];
    r20 = r0;
    r0 = [r0 componentsJoinedByString:@""];
    r0 = [r0 retain];
    [self performSelector:NSSelectorFromString(r0) withObject:zero_extend_64(0x0)];
    [r0 release];
    [r20 release];
    if (___stack_chk_guard != *(r31 + 0x28)) {
            __stack_chk_fail();
    }
    return;
}
```

可以看出，伪代码有着充分的信息，可以进行静态分析推导，判断代码片段是否调用了私有 API。

然而这种方法也有漏洞，拼接的字符串由[服务器](https://cloud.tencent.com/product/cvm?from=10680)下发，可以避开检查。

# 3、循环引用

## 3.1 造成循环引用的方式有哪些？

> 1. block
> 1. delegate
> 1. NSTimer
> 1. // Swift 方法嵌套

## 3.2 分别有什么解决方法？

> block内循环引用解决方法:
>
> > 1. 借助weak解决循环引用
> > 1. 借助__block解决循环引用
> > 1. "将强引用对象作为block参数传入"解决循环引用
> > 1. 借助NSProxy解决循环引用
>
> delegate造成循环引用解决方法：
>
> > delegate 使用weak修饰符修饰

## 3.3 NSTimer的循环引用一般用什么方式解决？

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

# 4、NSProxy 

## 4.1 NSProxy了解吗？

```c++
NSProxy is an abstract superclass defining an API for objects that act as stand-ins for other objects or for objects that don’t exist yet. Typically, a message to a proxy is forwarded to the real object or causes the proxy to load (or transform itself into) the real object. Subclasses of NSProxy can be used to implement transparent distributed messaging (for example, NSDistantObject) or for lazy instantiation of objects that are expensive to create.

翻译：NSProxy是一个抽象的超类，它定义了一个对象的API，用来充当其他对象或者一些不存在的对象的替身。通常，发送给Proxy的消息会被转发给实际对象，或使Proxy加载（转化为）实际对象。 NSProxy的子类可以用于实现透明的分布式消息传递(例如，NSDistantObject)，或者用于创建开销较大的对象的惰性实例化。
```



## 4.2 基本用途

> 1. 解决NSTimer的循环引用
> 2. 消息转发
> 3. 模拟多继承

## 4.3 NSProxy 与NSObject的区别

虽然NSProxy和class NSObject都定义了`-forwardInvocation:`和`-methodSignatureForSelector:`，但这两个方法并没有在protocol NSObject中声明；两者对这俩方法的调用逻辑更是完全不同。

对于class NSObject而言，接收到消息后先去自身的方法列表里找匹配的selector，如果找不到，会沿着继承体系去superclass的方法列表找；如果还找不到，先后会经过`+resolveInstanceMethod:`和`-forwardingTargetForSelector:`处理，处理失败后，才会到`-methodSignatureForSelector:`/`-forwardInvocation:`进行最后的挣扎.

但对于NSProxy，接收unknown selector后，直接回调`-methodSignatureForSelector:`/`-forwardInvocation:`，消息转发过程比class NSObject要简单得多。

相对于class NSObject，NSProxy的另外一个非常重要的不同点也值得注意：NSProxy会将自省相关的selector直接forward到`-forwardInvocation:`回调中，这些自省方法包括：

```objc
- (BOOL)isKindOfClass:(Class)aClass;
- (BOOL)isMemberOfClass:(Class)aClass;
- (BOOL)conformsToProtocol:(Protocol *)aProtocol;
- (BOOL)respondsToSelector:(SEL)aSelector;
```

简单来说，这4个selector的实际接收者realObject，而不是NSProxy对象本身。但另一方面，NSProxy并没有将performSelector系列selector也forward到`-forwardInvocation:`，换句话说，`[proxy performSelector:someSelector]`的真正处理者仍然是proxy自身，只是后续会将someSelector给forward到`-forwardInvocation`:回调，然后经由realObject处理。



# 5、NSObject

## 5.1 对象的引用计数一般存在什么地方？

> 经过优化的对象：
>
> 未经过优化的对象：
>
> 引用计数的存储位置分三种情况：
>
> * 特例：TaggedPointer对象：苹果会直接将其指针值作为引用计数返回
> * 开启isa优化的对象（NONPOINTER_ISA == 1）：引用计数存储在isa_t中的`shiftcls`字段中
> * 普通对象（NONPOINTER_ISA == 0） and  `shiftcls`字段越界的对象： 引用计数存储在SideTable中，通过对象的内存地址找到散列表周边保存的`retaincount`（最终值：`shiftcls` + `retaincount` + 1）
>



# 6、UIKit &Core Animation 

### Objective-C 语言

>  **Objective-C是面向对象的语言**
>
> Objective-C和Java C++一样，有封装，继承，多态，重用。但是它不像C++那样有重载操作法、模版和多继承，也没有Java的垃圾回收机制。
>
> **Objective-C的优点**
>
> Objective-C语言有C++ Java等面向对象的特点，那是远远不能体现它的优点的。Objective-C的优点是它是动态的。动态能力有三种：
>
> - **动态类**——运行时确定类的对象
> - **动态绑定**——运行时确定要调用的方法
> - **动态加载**——运行时为程序加载新的模块





## 6.1 UIKit 、Core Animation分别是用来做什么的？

Core Animation是一个复合引擎，它的职责就是尽可能快地组合屏幕上不同的可视内容，这个内容是被分解成独立的图层，存储在一个叫做图层树的体系之中。于是这个树形成了UIKit以及在iOS应用程序当中你所能在屏幕上看见的一切的基础。

UIKit 是在Core Animation之上做了一层基于iOS 端事件响应以及简单动画等一系列API的封装，避免开发者频繁使用底层API加重CPU负担。

## 6.2 为什么要同时存在这两种框架？

> 原因在于要做职责分离，这样也能避免很多重复代码。在iOS和Mac OS两个平台上，事件和用户交互有很多地方的不同，基于多点触控的用户界面和基于鼠标键盘有着本质的区别，这就是为什么iOS有UIKit和UIView，但是Mac OS有AppKit和NSView的原因。他们功能上很相似，但是在实现上有着显著的区别
>
> 

# 7、编译原理

## 7.1 应用的编译过程能说一下吗？

以（包含Swift文件与Pods库）的项目为例：

包含Swift文件与Pods库项目的编译可以分成三个大的过程：编译每个 pod 、编译整个 **Pod** 工程和编译主工程。

- **第一个过程：Xcodeproj 工具会为每个 pod 单独创建一个 target ，具体编译过程如下：**

  > 1. 1. 处理 **info.plist** 文件
  >    2. **clang** 编译所有 **.m/.c/.mm** 源文件为 **.o** 目标文件
  >    3. **Lb** 命令将 **.o** 文件链接成为一个 **Framework**
  >    4. 拷贝所有 **.h** 头文件与 **.bundle** 资源文件到 **Xyz.framework/Headers** 目录
  >    5. **codesign** 对 **Framework** 进行代码签名

- **第二个过程：比较简单，主要就是生成了一个 Pod 工程的静态库**。

- **第三个过程：编译主工程的过程，步骤较多：

  > 1. 1. 写入一些辅助脚本文件到临时文件夹
  >    2. 创建 **app** 目录
  >    3. 检查 **Entitlements** 文件
  >    4. 运行 **CocoaPods** 自定义脚本检查 **Manifest.lock**
  >    5. 编译所有 **swift** 文件，然后 **merge** 成一个 **swiftmodule**
  >    6. 拷贝 **Xyz-Swift.h** 头文件
  >    7. 编译 **.xcdatamodeld** 数据模型文件
  >    8. 编译 **.m/.c/.mm** 源文件
  >    9. 拷贝 **.swiftmodule** 文件
  >    10. 拷贝 **.swiftdoc** 文件
  >    11. 链接 **.o** 目标文件
  >    12. 用 **ibtool** 编译 **xib/storyboard UI**文件
  >    13. 用 **actool** 编译 **AssetCatalog** 资源文件
  >    14. 处理 **info.plist** 文件
  >    15. 链接 **xibc/storyboardc** 文件
  >    16. 运行 **CocoaPods** 自定义脚本嵌入生成的 **Pod framework**
  >    17. 运行 **CocoaPods** 自定义脚本拷贝 **Pod** 里的资源
  >    18. 拷贝 **Swift** 标准库到 **App** 中（包体积增大约**18M**）（**iOS12** 以上应该已经内置标准库，不需要再拷贝了）
  >    19. **touch** 生成 **.app** 文件
  >    20. 对 **App** 进行代码签名

总结：

> 1. 编译信息写入辅助文件，创建文件架构 .app 文件
> 2. 处理文件打包信息
> 3. 执行 CocoaPod 编译前脚本，checkPods Manifest.lock
> 4. 编译.m文件，使用 CompileC 和 clang 命令
> 5. *链接需要的 Framework*
> 6. 编译 xib
> 7. 拷贝 xib ，资源文件
> 8. 编译 ImageAssets
> 9. 处理 info.plist
> 10. 执行 CocoaPod 脚本
> 11. 拷贝标准库
> 12. 创建 .app 文件和签名
>
> 不理解的地方可参考：[App程序编译的完整流程](https://acefish.github.io/15538518127841.html)

## 7.2 针对某一个源文件的编译，它又经过哪几步呢？

### 1.2.1 预处理

主要完成以下操作：

> 1. import相关头文件
> 2. 完成宏替换
> 3. 处理以"#"开头的预处理命令
> 4. 删除注释

### 1.2.2 词法分析

主要是完成： 将预处理过的代码里切成一个个 Token，比如大小括号，等于号还有字符串等

### 1.2.3 语法分析

> 1. 在 Clang 中由 Parser 和 Sema 两个模块配合完成
> 2. 验证语法是否正确（比如语句末尾缺少分号）
> 3. 根据当前语言的语法，生成语义节点，并将所有节点组合成抽象语法树（AST）

### 1.2.4 静态分析

> 1. 通过语法树进行代码静态分析，找出非语法性错误
> 2. 模拟代码执行路径，分析出control-flow graph（CFG）

### 1.2.5 code-gen(中间代码生成)

> - CodeGen 负责将语法树从顶至下遍历，翻译成 LLVM IR。LLVM IR 是 Frontend 的输出，也是 LLVM Backend 的输入，前后端的桥接语言
>
> - 与 Objective-C Runtime 桥接：
>
>   > 主要内容包括：
>   >
>   > 1. Class / Meta Class / Protocol / Category 内存结构生成，并存放在指定 section 中（如 Class：_DATA，_objc_classrefs）
>   > 2. Method / Ivar / Property 内存结构的生成
>   > 3. 组成 method_list / ivar_list /property_list 并填入Class
>   > 4. 将语法树中的 ObjCMessageExpr 翻译成相应版本的 objc_msgSend，对 super 关键字的调用翻译成 objc_msgSendSuper
>   > 5. 根据修饰符 strong / weak / copy / atomic 合成 @property 自动实现的 setter / getter
>   > 6. ARC：分析对象引用关系，将 objc_storeStrong / objc_storeWeak 等 ARC 代码插入
>   > 7. 将 ObjCAutoreleasePoolStmt 转译成 objc_autoreleasePoolPush/Pop
>   > 8. 实现自动调用 [super dealloc]
>   > 9. 为每个拥有 ivar 的Class 合成 .cxx_destructor 方法来自动释放类的成员变量，代替 MRC 的 "self.xxx = nil"
>   > 10. Non-Fragile ABI：为每个 Ivar 合成 OBJC_IVAR_$_ 偏移值常量
>   > 11. 存取 Ivar 的语句 (*ivar = 123; int a = \*ivar;) 转写成 base + OBJC_IVAR\*$* 的形式
>   > 12. 处理 @synthesize
>   > 13. ⽣成 block_layout 的数据结构
>   > 14. 变量的 capture (__block / __weak)
>   > 15. ⽣成 _block_invoke 函数



# 8、能说一下使用xib过程遇到的问题吗？

https://www.jianshu.com/p/7d209c9761ba

# 9、Https

## 9.1 能简单描述一下Https建立连接的过程吗？

https://juejin.cn/post/7136170814990188558

# 10、说一下LRU算法的数据结构吧？针对LRU结构的常见操作，它们的时间复杂度分别是多少？（查询、插入、删除）

# 11、算法：两个子视图，寻找它们最近的共同父视图。

# 12、算法：检查两个单链表是否相交，如果相交返回相交节点。

# 13、SDK

## 13.1 怎么设计一个SDK？

## 13.2 SDK的调用接口，如果需要传入业务自己的实现方法，一般用什么方式？

## 13.3 怎么保证SDK内部函数间的耦合低？

## 定义什么样的函数接口的参数可以保证函数间耦合性尽量低？

# 14、单元测试

## 14.1 一般用什么方式实现单元测试？

## 14.2 你认为单元测试的重要指标有哪些？

## 14.3 你写的SDK的测试覆盖率有多少?怎么保证尽量高的覆盖率？

## 14.4、单元测试一般以什么粒度书写？如果写的过程中发现某个函数无法写测试用例怎么办？

## 14.5、遇到网络请求接口的时候，怎么进行单元测试？有什么方式可以拦截全局的网络请求？

# 15、hook

有什么方式可以直接hook全局的函数接口？

# 16、GCD

## 16.1  GCD 底层是怎么进行线程管理的？

# 17、Runloop 

## 17.1 对 Runloop 的理解

> Runloop 是个死循环吗？Runloop是怎么实现休眠的？为什么休眠后还能唤醒？

## 17.2 runloop使用场景

你都在哪些场景下使用过 Runloop ? 

## 17.3 Runloop 怎么监控卡顿？

# 18、NSTimer 

## 18.1 NSTimer是什么实现的？

## 18.2 为什么它会不准？一般什么情况会导致它不准？

# 19、算法：验证一个 字符串 是否是有效的 IPV4 地址。

