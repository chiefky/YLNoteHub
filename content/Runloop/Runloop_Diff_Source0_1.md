# source0 和 source1 的区别

<img src="/Users/tangh/yuki/博客/文章仓库/YLNoteHub/content/images/runloop_CFRunLoopSourceContext.png" alt="runloop_CFRunLoopSourceContext" style="zoom:60%;" />



Source0 解读：

* source0 仅包含一个回调函数（perform），它并不能主动唤醒 run loop（进入休眠的 run loop 仅能通过 mach port 和 mach_msg 来唤醒）。
* 你需要先调用 CFRunLoopSourceSignal(rls) 将这个 source 标记为待处理，然后手动调用 CFRunLoopWakeUp(rl) 来唤醒 run loop（CFRunLoopWakeUp 函数内部是通过 run loop 实例的 _wakeUpPort 成员变量来唤醒 run loop 的）
* 唤醒后的 run loop 继续执行 __CFRunLoopRun 函数内部的外层 do while 循环来执行 timers（执行到达执行时间点的 timer 以及更新下次最近的时间点） 和 sources 以及 observer 回调 run loop 状态，其中通过调用 __CFRunLoopDoSources0 函数来执行 source0 事件，执行过后的 source0 会被 __CFRunLoopSourceUnsetSignaled(rls) 标记为已处理，后续 run loop 循环中不会再执行标记为已处理的 source0。
* source0 不同于不重复执行的 timer 和 run loop 的 block 链表中的 block 节点，source0 执行过后不会自己主动移除，不重复执行的 timer 和 block 执行过后会自己主动移除，执行过后的 source0 可手动调用 CFRunLoopRemoveSource(CFRunLoopGetCurrent(), rls, kCFRunLoopDefaultMode) 来移除。

参考链接：https://juejin.cn/post/6913094534037504014

