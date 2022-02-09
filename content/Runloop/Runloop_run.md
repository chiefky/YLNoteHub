# RunLoop的正确使用

注意：

1. runloop 必须添加 source 或 timer 后才能 run，否则直接退出
2. runloop 相关 Run 方法会开启事件循环，同时 block 住 Run 后续的代码执行，直到 Runloop 退出

系统提供开启Runloop的接口有：

> * \- (void)run; 
> * \- (void)runUntilDate:(NSDate *)limitDate;
> * \- (BOOL)runMode:(NSRunLoopMode)mode beforeDate:(NSDate *)limitDate;

### 开启runloop

* run 方法解析：
  * 运行模式为默认的NSDefaultRunLoopMode模式
  * 没有超时限制，因为无条件运行
  * 接口文档上标明：<font color='red'>该方法本质是无限调用 runMode:beforeDate: 方法</font>
  * 不建议使用：因为这个接口会导致Run Loop永久性的运行在NSDefaultRunLoopMode模式，即使使用 CFRunLoopStop(runloopRef) 也无法停止Run Loop的运行，最终造成内存泄露。

* runUntilDate:方法解析：
  * limitDate：超时时间
  * 运行在NSDefaultRunLoopMode模式
  * 接口文档上标明：<font color='red'>该方法本质是无限调用 runMode:beforeDate: 方法</font>
  * 不建议使用：CFRunLoopStop(runloopRef) 也无法停止Run Loop的运行，因此最好设置一个合理的 Run Loop 运行时间

*  runMode: beforeDate: 方法解析：
   * 表示的是 runloop 的单次调用
   * 在非Timer事件触发、显式调用 CFRunLoopStop、到达limitDate后会退出返回
   * 如果仅是Timer事件触发并不会让Run Loop退出返回
   * 如果因为没有 source 或 timer 造成的直接退出，返回 NO
   * 如果是 PerfromSelector 事件或者其他Input Source事件触发处理后，Run Loop会退出返回 YES

### 终止runloop

* 手动终止：`CFRunLoopStop()`

  > **使用 CFRunLoopStop() 无法停止 - (void)run 与 - (void)runUntilDate:(NSDate *)limitDate 的原因：**
  >
  > CFRunLoopStop() 方法只会结束当前的 runMode:beforeDate: 调用，而不会结束后续的调用。 这也就是为什么 Runloop 的文档中说 CFRunLoopStop() 可以 exit(退出) 一个 runloop，而在 run 等方法的文档中又说这样会导致 runloop 无法 terminate(终结)

* timer开启的runloop只能通过`[timer invalidate];`方法终止
* 可以通过移除事件源使runloop终止，线程销毁时runloop自动销毁