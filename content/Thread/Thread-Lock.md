# 1. 概念

- 锁，是保证线程安全常见的同步工具。
- 锁，是一种非强制的机制，每一个线程在访问数据或者资源前，要先获取(Acquire) 锁，并在访问结束之后释放(Release)锁。如果锁已经被占用，其它试图获取锁的线程会等待，直到锁重新可用。



锁：在计算机科学中，锁是一种同步机制，用于在存在多线程的环境中实施对资源的访问限制。你可以理解成它用于排除并发的一种策略！你可以理解为为了防止多线程访问下资源的抢夺，保持线程同步的方式

# 2. 作用

- 将线程不安全的代码 “锁” 起来。保证一段代码或者多段代码操作的原子性，保证多个线程对同一个数据的访问 同步 **(Synchronization)**。

另一种描述：

- 防止在多线程（多任务）的情况下对共享资源（临界资源）的脏读或者脏写。

### 属性设置 **atomic**

- atomic 的底层实现，老版本是自旋锁（OSSpinLock），iOS10开始，由于优先级翻转的问题，开始改用优化了性能的互斥锁（os_unfair_lock）

# 3. 锁的分类

- 自旋锁。例如：OSSpinLock 等
- 互斥锁：例如：pthread_mutex 等

## 3.1 两种锁的特点对比

- 相同点：都能保证同一时间只有一个线程访问共享资源。都能保证线程安全。
- 不同点： 

- - 互斥锁：如果共享数据已经有其他线程加锁了，线程会进入休眠状态(sleep-waiting)等待锁。一旦被访问的资源被解锁，则等待资源的线程会被唤醒。

  - 自旋锁：如果共享数据已经有其他线程加锁了，线程会以死循环的方式(busy-waiting)等待锁，一旦被访问的资源被解锁，则等待资源的线程会立即执行。


> 自旋锁看起来是比较耗费 cpu 的，然而在互斥临界区计算量较小的场景下，它的效率远高于其它的锁。
>   <font color='red 原因：自旋锁一直处于 running 状态，减少了线程切换上下文的消耗。</font>

注意事项：

> 使用自旋锁时要注意：
>
> - 由于自旋时不释放CPU，因而持有自旋锁的线程应该尽快释放自旋锁，否则等待该自旋锁的线程会一直在哪里自旋，这就会浪费CPU时间。
> - 持有自旋锁的线程在sleep之前应该释放自旋锁以便其他线程可以获得该自旋锁。内核编程中，如果持有自旋锁的代码sleep了就可能导致整个系统挂起。
>
>   使用任何锁都需要消耗系统资源（内存资源和CPU时间），这种资源消耗可以分为两类：
>
> ​    1.建立锁所需要的资源
>
> ​    2.当线程被阻塞时所需要的资源

## 3.2 两种锁的加锁原理

互斥锁：线程会从sleep（加锁）——>running（解锁），过程中有上下文的切换(主动出让时间片，线程休眠，等待下一次唤醒)，cpu的抢占，信号的发送等开销。

自旋锁：线程一直是running(加锁——>解锁)，死循环（忙等 do-while）检测锁的标志位，机制不复杂。

# 4. 递归锁

- 递归锁也称为可重入锁。

- 互斥锁可以分为非递归锁/递归锁两种：

- - 递归锁：同一个线程可以重复获取递归锁，不会死锁; 
  - 非递归锁：同一个线程重复获取非递归锁，则会产生死锁。

🌰代码：

```objective-c
- (void)testLock{
  if(_count>0){ 
   @synchronized (obj) {
     _count = _count - 1;
     [self testLock];
   }
  }
 }
```

死锁的条件:

1. 互斥条件：一个资源每次只能被一个进程使用；

2. 请求与保持条件：一个进程因请求资源而阻塞时，对已获得的资源保持不放；

3. 不可剥夺条件：进程已获得的资源，在末使用完之前，不能强行剥夺；

4. 循环等待条件：若干进程之间形成一种头尾相接的循环等待资源关系；

# 5. 常用锁

> 1. OSSpinLock 自旋锁 （ os_unfair_lock iOS10之后替代OSSPinLock的锁，解决了优先级反转的问题）
> 2. dispatch_semaphore 信号量实现加锁（GCD）
> 3. pthread_mutex 互斥锁（C语言）
> 4. NSLock 对象锁 
> 5. NSCondition  
> 6. NSRecursiveLock 递归锁
> 7. NSConditionLock 条件锁  
> 8. @synchronized 关键字加锁 ，递归锁
> 9. pthread_rwlock（读写锁）

**加解锁耗时对比图：**

<img src="/Users/tangh/yuki/博客/文章仓库/YLNoteHub/content/Thread/image/Thread_lock_0.png" style="zoom:80%;" />

## 5.1 OSSpinLock

OSSpinLock 是一种自旋锁。它的特点是在线程等待时会一直轮询，处于忙等状态。

使用方法

使用需导入头文件
 \#import <libkern/OSAtomic.h>

- OSSpinLock oslock = OS_SPINLOCK_INIT;  //锁的初始化：默认值为 0，在 locked 状态时就会大于 0，unlocked状态下为 0
- OSSpinLockLock(&oslock);    加锁
- OSSpinLockUnlock(&oslock);  解锁
- OSSpinLockTry(&oslock);   尝试加锁，可以加锁则立即加锁并返回YES，反之返回NO

### 忙等这种自旋锁的实现原理：

bool lock = false; *//* 一开始没有锁上，任何线程都可以申请锁 
 do { 
   while(test_and_set(&lock); *// test_and_set* 是一个原子操作。如果 *lock* 为 *true* 就一直死循环，相当于申请锁。如果为 *false*，则挂上锁，这样别的线程就无法获得锁
     Critical section *//* 临界区
   lock = false; *//* 相当于释放锁，这样别的线程可以进入临界区
     Reminder section *//* 不需要锁保护的代码    
 }

bool test_and_set (bool *target) { 
   bool rv = *target; 
   *target = TRUE; 
   return rv;
 }

注意：

如果临界区的执行时间过长，最好不要使用自旋锁。线程会在多种情况下会退出自己的时间片，其中一种是用完了时间片的时间，被操作系统强制抢占。除此以外，当线程进行 I/O 操作，或进入睡眠状态时，都会主动让出时间片。显然在 while 循环中，线程处于忙等状态，白白浪费 CPU 时间，最终因为超时被操作系统抢占时间片。如果临界区执行时间较长，比如是文件读写，这种忙等是毫无必要的。

**存在的问题**

[OSSpinLock 不再安全](https://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)，主要原因发生在低优先级线程拿到锁时，高优先级线程进入忙等(busy-wait)状态，消耗大量 CPU 时间，从而导致低优先级线程拿不到 CPU 时间，也就无法完成任务并释放锁。这种问题被称为优先级反转。

**os_unfair_lock**：

- 为了解决优先级反转的问题，替换已弃用的OSSpinLock
- 必须使用OS_UNFAIR_LOCK_INIT来初始化
- 加锁和解锁必须在相同的线程，否则会中断进程
- __IOS_AVAILABLE(10.0)

使用方法

- - os_unfair_lock_t unfairLock = &(OS_UNFAIR_LOCK_INIT); 
  - os_unfair_lock_lock(unfairLock); 
  - os_unfair_lock_unlock(unfairLock);

### 优先级反转**(Priority Inversion)**

什么情况叫做优先级反转？

wikipedia 上是这么定义的：

优先级倒置，又称优先级反转、优先级逆转、优先级翻转，是一种不希望发生的任务调度状态。在该种状态下，一个高优先级任务间接被一个低优先级任务所抢先(preemtped)，使得两个任务的相对优先级被倒置。 这往往出现在一个高优先级任务等待访问一个被低优先级任务正在使用的临界资源，从而阻塞了高优先级任务；同时，该低优先级任务被一个次高优先级的任务所抢先，从而无法及时地释放该临界资源。这种情况下，该次高优先级任务获得执行权。



举例解释一下：

有：高优先级任务A / 次高优先级任务B / 低优先级任务C / 资源Z 。

A 等待 C 执行后的 Z，

而 B 并不需要 Z，抢先获得时间片执行。

C 由于没有时间片，无法执行(优先级相对没有B高)。



这种情况造成 A 在C 之后执行,C在B之后，间接的高优先级A在次高优先级任务B 之后执行, 使得优先级被倒置了。（假设： A 等待资源时不是阻塞等待，而是忙循环，则可能永远无法获得资源。此时 C 无法与 A 争夺 CPU 时间，从而 C 无法执行，进而无法释放资源。造成的后果，就是 A 无法获得 Z 而继续推进。）

为什么忙等会导致低优先级线程拿不到时间片？

- 现代操作系统在管理普通线程时，通常采用时间片轮转算法(Round Robin，简称 RR)。每个线程会被分配一段时间片(quantum)，通常在 10-100 毫秒左右。当线程用完属于自己的时间片以后，就会被操作系统挂起，放入等待队列中，直到下一次被分配时间片。

而 OSSpinLock 忙等的机制，就可能造成高优先级一直 running ，占用 cpu 时间片。而低优先级任务无法抢占时间片，变成迟迟完不成，不释放锁的情况。



### **优先级反转的解决方案**

关于优先级反转一般有以下三种解决方案：

- 优先级继承

​    优先级继承，故名思义，是将占有锁的线程优先级，继承等待该锁的线程高优先级，如果存在多个线程等待，就取其中之一最高的优先级继承。

- 优先级天花板

​    优先级天花板，则是直接设置优先级上限，给临界区一个最高优先级，进入临界区的进程都将获得这个高优先级。

​    如果其他试图进入临界区的进程的优先级，都低于这个最高优先级，那么优先级反转就不会发生。

- 禁止中断

​    禁止中断的特点，在于任务只存在两种优先级：可被抢占的 / 禁止中断的 。

​    前者为一般任务运行时的优先级，后者为进入临界区的优先级。

​    通过禁止中断来保护临界区，没有其它第三种的优先级，也就不可能发生反转了。

### 为什么使用其它的锁，可以解决优先级反转？

原因在于，其它锁（比如 pthread_mutex 和 dispatch_semaphore ）出现优先级反转后，高优先级的任务不会忙等。因为处于等待状态的高优先级任务，没有占用时间片，所以低优先级任务一般都能进行下去，从而释放掉锁。

### 线程调度

- 无论多核心还是单核，我们的线程运行总是 "并发" 的。

- 当 cpu 数量大于等于线程数量，这个时候是真正并发，可以多个线程同时执行计算。

- 当 cpu 数量小于线程数量，总有一个 cpu 会运行多个线程，这时候"并发"就是一种模拟出来的状态。操作系统通过不断的切换线程，每个线程执行一小段时间，让多个线程看起来就像在同时运行。这种行为就称为 "线程调度（Thread Schedule）"。

### 线程状态

- 在线程调度中，线程至少拥有三种状态 : 运行(Running)、就绪(Ready)、等待(Waiting)。

- 处于 Running的线程拥有的执行时间，称为 时间片(Time Slice)，时间片 用完时，进入Ready状态。如果在Running状态，时间片没有用完，就开始等待某一个事件（通常是 IO 或 同步 ），则进入Waiting状态。

- 如果有线程从Running状态离开，调度系统就会选择一个Ready的线程进入 Running 状态。而Waiting的线程等待的事件完成后，就会进入Ready状态。

## 5.2 dispatch_semaphore（信号量）

- dispatch_semaphore 是信号量，当信号总量设为 1 时也可以当作锁。
- 在没有等待情况出现时，它的性能比 pthread_mutex 还要高.
- 一旦有等待情况出现时，性能就会下降许多。相对于 OSSpinLock 来说，它的优势在于等待时不会消耗 CPU 资源。

**主要接口介绍**

- 创建信号：dispatch_semaphore_create(long value) 传入值必须 >=0, 若传入为 0 则阻塞线程并等待timeout，时间到后会执行其后的语句

- 等待信号到达：dispatch_semaphore_wait(dispatch_semaphore_t dsema, dispatch_time_t timeout) 使得signal值-1

- 发送信号，解除等待状态：dispatch_semaphore_signal(dispatch_semaphore_t deem)使得signal值+1

### 底层实现原理

dispatch_semaphore_t 的实现最终会调用到 sem_wait 方法，这个方法在 glibc 中被实现如下：

```objective-c
int sem_wait (sem_t *sem) {
  int *futex = (int *) sem;
  if (atomic_decrement_if_positive (futex) > 0)
    return 0;
  int err = lll_futex_wait (futex, 0);
    return -1;
)
```



- 首先会把信号量的值减一，并判断是否大于零。如果大于零，说明不用等待，所以立刻返回。具体的等待操作在 lll_futex_wait 函数中实现，**lll** 是 low level lock 的简称。这个函数通过汇编代码实现，调用到 SYS_futex 这个系统调用，使线程进入睡眠状态，主动让出时间片，这个函数在互斥锁的实现中，也有可能被用到。

- 主动让出时间片并不总是代表效率高。让出时间片会导致操作系统切换到另一个线程，这种上下文切换通常需要 10 微秒左右，而且至少需要两次切换。如果等待时间很短，比如只有几个微秒，忙等就比线程睡眠更高效。

- 可以看到，自旋锁和信号量的实现都非常简单，这也是两者的加解锁耗时分别排在第一和第二的原因。再次强调，加解锁耗时不能准确反应出锁的效率(比如时间片切换就无法发生)，它只能从一定程度上衡量锁的实现复杂程度。

### 信号量机制

信号量中，二元信号量，是一种最简单的锁。只有两种状态，占用和非占用。二元信号量适合唯一一个线程独占访问的资源。而多元信号量简称 信号量(Semaphore)。

### 信号量和互斥量的区别

参考：

1. [互斥量和信号量的区别](https://www.cnblogs.com/lbsx/archive/2009/08/03/1537698.html)
2. [信号量和互斥锁的区别](https://www.jianshu.com/p/c6ba8bcc22bc)

- 信号量是允许并发访问的，也就是说，允许多个线程同时执行多个任务。信号量可以由一个线程获取，然后由不同的线程释放

- 互斥量只允许一个线程同时执行一个任务。也就是同一个线程获取，同一个线程释放

- 可以在不同线程获取/释放同一个互斥锁，但互斥锁本来就用于同一个线程中上锁和解锁。这里的意义更多在于代码使用的层面。
- 关键在于，理解信号量可以允许 N 个信号量允许 N 个线程并发地执行任务。

## 5.3 pthread_mutex（互斥锁）

- pthread 表示 POSIX thread，定义了一组跨平台的线程相关的 API

- pthread_mutex 表示互斥锁。互斥锁的实现原理与信号量非常相似，不是使用忙等，而是阻塞线程并睡眠，需要进行上下文切换

- 对于 pthread_mutex 来说，它的用法和之前没有太大的改变，比较重要的是锁的类型，可以有 PTHREAD_MUTEX_NORMAL、PTHREAD_MUTEX_ERRORCHECK、PTHREAD_MUTEX_RECURSIVE 等等

- 一个线程只能申请一次锁，也只能在获得锁的情况下才能释放锁，多次申请锁或释放未获得的锁都会导致崩溃。假设在已经获得锁的情况下再次申请锁，线程会因为等待锁的释放而进入睡眠状态，因此就不可能再释放锁，从而导致死锁

- pthread_mutex 本身拥有设置协议的功能，通过设置它的协议，来解决优先级反转:

   pthread_mutexattr_setprotocol(pthread_mutexattr_t *attr, int protocol)

   其中协议类型包括以下几种：

- - PTHREAD_PRIO_NONE：线程的优先级和调度不会受到互斥锁拥有权的影响

- - PTHREAD_PRIO_INHERIT：当高优先级的等待低优先级的线程锁定互斥量时，低优先级的线程以高优先级线程的优先级运行。这种方式将以继承的形式传递。当线程解锁互斥量时，线程的优先级自动被降到它原来的优先级。该协议就是支持优先级继承类型的互斥锁，它不是默认选项，需要在程序中进行设置。

- - PTHREAD_PRIO_PROTECT：当线程拥有一个或多个使用 PTHREAD_PRIO_PROTECT初始化的互斥锁时，此协议值会影响其他线程（如 thrd2）的优先级和调度。thrd2 以其较高的优先级或者以thrd2拥有的所有互斥锁的最高优先级上限运行。基于被thrd2拥有的任一互斥锁阻塞的较高优先级线程对于 thrd2的调度没有任何影响。

​    设置协议类型为 PTHREAD_PRIO_INHERIT ，运用优先级继承的方式，可以解决优先级反转的问题

- 在 iOS 中使用的 NSLock,NSRecursiveLock 等都是基于pthread_mutex 做实现的

互斥锁的常用接口如下**:**

- 使用需导入头文件： #import <pthread.h>

- pthread_mutexattr_t attr;
- pthread_mutexattr_init(&attr);   //初始化attr并且给它赋予默认
- pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_NORMAL); //设置锁类型，这边是设置为标准互斥锁

- pthread_mutex_t pLock;
- pthread_mutex_init(&pLock, &attr); // 初始化的时候带入参数

- pthread_mutex_lock(&mutex);   // 申请锁

  // 临界区

- pthread_mutex_unlock(&mutex); // 释放锁

- pthread_mutex_destroy(&pLock) // 销毁锁对象
- pthread_mutexattr_destroy(&attr); //销毁一个属性对象，在重新进行初始化之前该结构不能重新使用

### 互斥锁的实现原理

- 互斥锁在申请锁时，调用了 pthread_mutex_lock 方法，它在不同的系统上实现各有不同，有时候它的内部是使用信号量来实现，即使不用信号量，也会调用到 lll_futex_wait 函数，从而导致线程休眠。

- 如果临界区很短，忙等的效率也许更高，所以在有些版本的实现中，会首先尝试一定次数(比如 1000 次)的 test_and_test，这样可以在错误使用互斥锁时提高性能。

## 5.4 **NSLock**

### （类型为 **PTHREAD_MUTEX_ERRORCHECK** 的互斥锁）

- NSLock 是 Objective-C 以对象的形式暴露给开发者的一种锁

- NSLock 只是在内部封装了一个 pthread_mutex，属性为 PTHREAD_MUTEX_ERRORCHECK，它会损失一定性能换来错误提示。

- 它的实现非常简单，通过宏，定义了 lock 方法:

```objective-c
#define  MLOCK \

- (void) lock\

{\

 int err = pthread_mutex_lock(&_mutex);\

 // 错误处理 ……

}
```

- 这里使用宏定义的原因是：OC 内部还有其他几种锁，他们的 lock 方法都是一模一样，仅仅是内部 pthread_mutex 互斥锁的类型不同。通过宏定义，可以简化方法的定义。

- NSLock 比 pthread_mutex 略慢的原因在于它需要经过方法调用，同时由于缓存的存在，多次方法调用不会对性能产生太大的影响。

- 对象锁均实现了NSLocking协议：

```objc
@protocol NSLocking
- (void)lock;
- (void)unlock;

@end
```

常用接口如下**:**

- trylock：能加锁返回YES并执行加锁操作，相当于lock，反之返回NO

- lockBeforeDate：表示会在传入的时间内尝试加锁，若能加锁则执行加锁操作并返回YES，反之返回NO

## **5**.5 **NSCondition**

- NSCondition 是最基本的条件锁，底层是通过条件变量(condition variable) pthread_cond_t 来实现，实际上封装了一个互斥锁和条件变量。手动控制线程wait和signal

- 条件变量有点像信号量，提供了线程阻塞与信号机制，因此可以用来阻塞某个线程，并等待某个数据就绪，随后唤醒线程，比如常见的生产者-消费者模式。

- 锁实现了NSLocking协议

常用接口如下**:**

- wait：进入等待状态

- waitUntilDate：让一个线程等待一定的时间

- signal：唤醒一个等待的线程

- broadcast：唤醒所有等待的线程

注意问题：

- 由于broadcast可以唤醒所有被-wait方法阻塞的线程，所以在任意地方调用broadcast方法都可能影响到这里。

- 根据苹果官方文档，-signal 方法本身就不完全保证是准确的，会存在被wait的线程没有调用-signal方法，但是被其他线程调用 -signal 方法被唤醒的情况

- 就算被wait的线程的唤醒时机没有问题，但是在被wait的线程被唤醒到执行后面代码期间，程序状态可能会发生变化，这也是一个风险项。

### 条件变量

  在线程间的同步中，有这样一种情况： 线程 A 需要等条件 C 成立，才能继续往下执行.现在这个条件不成立，线程 A 就阻塞等待. 而线程 B 在执行过程中，使条件 C 成立了，就唤醒线程 A 继续执行。

对于上述情况，可以使用条件变量来操作。

- 条件变量，类似信号量，提供线程阻塞与信号机制，可以用来阻塞某个线程，等待某个数据就绪后，随后唤醒线程

- 一个条件变量总是和一个互斥量搭配使用

- NSCondition其实就是封装了一个互斥锁和条件变量，互斥锁的lock/unlock方法和后者的wait/signal统一封装在 NSCondition对象中，暴露给使用者

- 用条件变量控制线程同步，最为经典的例子就是 生产者-消费者问题。



### 生产者**-**消费者问题

生产者消费者问题,是一个著名的线程同步问题，该问题描述如下：

  有一个生产者在生产产品，这些产品将提供给若干个消费者去消费。要求让生产者和消费者能并发执行，在两者之间设置一个具有多个缓冲区的缓冲池，生产者将它生产的产品放入一个缓冲区中，消费者可以从缓冲区中取走产品进行消费，显然生产者和消费者之间必须保持同步，即不允许消费者到一个空的缓冲区中取产品，也不允许生产者向一个已经放入产品的缓冲区中再次投放产品。

### 条件变量 的使用

很多介绍 pthread_cond_t 的文章都会提到，它需要与互斥锁配合使用

生产者**-**消费者问题 🌰代码：

```objective-c
void consumer () { // 消费者

  pthread_mutex_lock(&mutex); // 互斥锁 上锁
  // wait 方法除了会被 signal 方法唤醒，有时还会被虚假唤醒，所以需要这里 while 循环中的判断来做二次确认。

  while (data == NULL) {
    pthread_cond_wait(&condition_variable_signal, &mutex); // 等待数据
  }

  // --- 有新的数据，以下代码负责处理 ↓↓↓↓↓↓
  // temp = data;
  // --- 有新的数据，以上代码负责处理 ↑↑↑↑↑↑
  pthread_mutex_unlock(&mutex);  // 互斥锁 解锁
}

void producer () {
  pthread_mutex_lock(&mutex);    // 互斥锁 上锁
  // 生产数据
  pthread_cond_signal(&condition_variable_signal); // 发出信号给消费者，告诉他们有了新的数据
  pthread_mutex_unlock(&mutex);  // 互斥锁 解锁
}

```

### 为什么要使用条件变量

- 信号量可以一定程度上替代 condition，但是互斥锁不行。在以上给出的生产者-消费者模式的代码中， pthread_cond_wait 方法的本质是锁的转移，消费者放弃锁，然后生产者获得锁，同理，pthread_cond_signal 则是一个锁从生产者到消费者转移的过程。

- 使用 condition 有一个好处，我们可以调用 pthread_cond_broadcast 方法通知所有等待中的消费者，这是使用信号量无法实现的。

- **NSCondition** 的加解锁过程与 NSLock 几乎一致，理论上来说耗时也应该一样(实际测试也是如此)

### 条件变量和信号量的区别

- 每个信号量有一个与之关联的值，发出时+1，等待时-1，任何线程都可以发出一个信号，即使没有线程在等待该信号量的值

- 可是对于条件变量，例如 pthread_cond_signal发出信号后，没有任何线程阻塞在 pthread_cond_wait上，那这个条件变量上的信号会直接丢失掉。

## 5.6 **NSRecursiveLock**（递归锁）

- NSRecursiveLock 和 @synchonized一样，是一个递归锁。

- 递归锁也是通过 pthread_mutex_lock 函数实现，在函数内部会判断锁的类型，如果显示是递归锁，就允许递归调用，仅仅将一个计数器加一，锁的释放过程也是同理。

- NSRecursiveLock 与 NSLock 的区别在于内部封装的 pthread_mutex_t 对象的类型不同，前者的类型为 PTHREAD_MUTEX_RECURSIVE。

- 锁实现了NSLocking协议

互斥锁的常用接口如下**:**

- pthread_mutexattr_t attr;
- pthread_mutexattr_init(&attr);   //初始化attr并且给它赋予默认

- pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE); //设置锁类型，这边是设置为递归锁

- pthread_mutex_t pLock;
- pthread_mutex_init(&pLock, &attr); // 初始化的时候带入参数

- pthread_mutex_destroy(&pLock) // 销毁锁对象
- pthread_mutexattr_destroy(&attr); //销毁一个属性对象，在重新进行初始化之前该结构不能重新使用

## **5.7****NSConditionLock**



- NSConditionLock 借助 NSCondition 来实现，它的本质就是一个生产者-消费者模型。“条件被满足”可以理解为生产者提供了新的内容。

- NSConditionLock 的内部持有一个 NSCondition 对象，以及 _condition_value 属性，在初始化时就会对这个属性进行赋值:

-  unlockWhenCondition 方法则是生产者，使用了 broadcast 方法通知了所有的消费者：它会先解锁，再修改 **condition** 参数的值。 并不是当 **condition** 符合某个件值去解锁。

  ```objective-c
  - (void) unlockWithCondition: (NSInteger)value {
    _condition_value = value;
    [_condition broadcast];
    [_condition unlock];
  }
  ```

-  lockWhenCondition 方法其实就是消费者方法：它与 unlockWithCondition: 不一样，不会修改 condition 参数的值，而是符合 condition 的值再上锁。

  ```objective-c
  - (void) lockWhenCondition: (NSInteger)value {
    [_condition lock];
    while (value != _condition_value) {
      [_condition wait];
    }
  }
  ```

  

- 锁实现了NSLocking协议

## 5.8 **@synchonized**（递归锁）

参考：

1. [关于 @synchronized，这儿比你想知道的还要多](http://yulingtianxia.com/blog/2015/11/01/More-than-you-want-to-know-about-synchronized/)（主要介绍了 @synchonized 的实现原理）
2. [正确使用多线程同步锁**@synchronized()**](https://www.jianshu.com/p/2dc347464188)

### 实现原理

- @synchronized 后面需要紧跟一个 OC 对象，它实际上是把这个对象当做锁来使用。这是通过一个哈希表来实现的，OC 在底层使用了一个互斥锁的数组(可以理解为锁池)，通过对对象去哈希值来得到对应的互斥锁。[具体介绍](https://blog.csdn.net/Deft_MKJing/article/details/82732833)

- 这个哈希表的实现和weak属性一样，是以对象为key，然后互斥锁的数组为value。

另一个版本：

- **synchronized**中传入的**object**的内存地址，被用作**key**，通过**hash map**对应的一个系统维护的递归锁。

更详细的版本：

- **synchronized**底层的实现原理是利用系统的**mutex** PTHREAD_MUTEX_RECURSIVE **Lock**实现。每一个可重入锁都会关联一个线程**ID**和一个锁状态**status**。

- 当一个线程请求锁时，会去检查锁状态，如果锁状态是**0**，代表该锁没有被占用，直接进行**CAS**操作获取锁，同时将线程**ID**替换成该线程**ID**。如果锁状态不是**0**，代表有线程在访问该方法。此时，如果线程**ID**是该线程**ID**，如果是可重入锁，会将**status**自增**1**，然后获取到该锁，进而执行相应的方法。如果是非重入锁，就会进入阻塞队列等待。

- 释放锁：

- - 可重入锁，每一次退出方法，就会将**status**减**1**，直至**status**的值为**0**，最后释放该锁。
  - 非可重入锁，线程退出方法，直接就会释放该锁。

注意

- **synchronized**是使用的递归**mutex**来做同步
- **@synchronized(nil)**不起任何作用
- 慎用**@synchronized(self)**

​    因为self 很可能会被外部对象调用和修改，被用作key来生成锁，容易产生两个公共锁交替使用的死锁情况 或 @synchronized(nil) 的情况。正确的做法是传入一个类内部维护的NSObject对象，而且这个对象是对外不可见的。

## **5.9 pthread_rwlock**（读写锁）

- 读写锁是用来解决文件读写问题的，读操作可以共享，写操作是排他的，读可以有多个在读，写只有唯一个在写，同时写的时候不允许读。

- - 当读写锁被一个线程以读模式占用的时候，写操作的其他线程会被阻塞，读操作的其他线程还可以继续进行
  - 当读写锁被一个线程以写模式占用的时候，写操作的其他线程会被阻塞，读操作的其他线程也被阻塞

常用接口介绍

- 初始化缺省属性的读写锁：pthread_rwlock_t rwlock = PTHREAD_RWLOCK_INITIALIZER;  

- 初始化指定属性的读写锁：pthread_rwlock_init(&rwlock, const pthread_rwlockattr_t * _Nullable __restrict)：返回值：0，表示成功；非0为表示错误码

- 阻塞式-写模式：pthread_rwlock_wrlock(&rwlock); 
- 非阻塞式-写模式：pthread_rwlock_trywrlock(&rwlock); 

- 阻塞式-读模式：pthread_rwlock_rdlock(&rwlock); 
- 非阻塞式-读模式：pthread_rwlock_tryrdlock(&rwlock); 

- 读模式或者写模式的解锁：pthread_rwlock_unlock(&rwlock);

-  销毁读写锁：pthread_rwlock_destroy(&rwlock)；

注意：

- 对于读数据比修改数据频繁的应用，用读写锁代替互斥锁可以提高效率。

   因为使用互斥锁时，即使是读出数据（相当于操作临界区资源）都要上互斥锁，而采用读写锁，则可以在任一时刻允许多个读出者存在，提高了更高的并发度，同时在某个写入者修改数据期间保护该数据，以免任何其它读出者或写入者的干扰。

## **5.10 NSDistributedLock** 分布式锁

- NSDistributedLock, 是 macOS 下的一种锁

- [苹果文档](https://link.segmentfault.com/?url=https%3A%2F%2Flinks.jianshu.com%2Fgo%3Fto%3Dhttps%3A%2F%2Flink.juejin.im%3Ftarget%3Dhttps%253A%252F%252Fdeveloper.apple.com%252Fdocumentation%252Ffoundation%252Fnsdistributedlock%253Flanguage%253Dobjc) 对于NSDistributedLock 的描述是:

​	A lock that multiple applications on multiple hosts can use to restrict access to some shared resource, such as a file

​	意思是说，它是一个用在多个主机间的多应用的锁，可以限制访问一些共享资源，例如文件。

- 按字面意思翻译，NSDistributedLock 应该就叫做 分布式锁。但是看概念和资料，在 [解决NSDistributedLock进程互斥锁的死锁问题(一)](https://link.segmentfault.com/?url=https%3A%2F%2Flinks.jianshu.com%2Fgo%3Fto%3Dhttps%3A%2F%2Flink.juejin.im%3Ftarget%3Dhttp%253A%252F%252Fwww.tanhao.me%252Fpieces%252F1731.html%252F) 里面看到，NSDistributedLock 更类似于文件锁的概念。 有兴趣的可以看一看 [Linux 2.6 中的文件锁](https://link.segmentfault.com/?url=https%3A%2F%2Flinks.jianshu.com%2Fgo%3Fto%3Dhttps%3A%2F%2Flink.juejin.im%3Ftarget%3Dhttps%253A%252F%252Fwww.ibm.com%252Fdeveloperworks%252Fcn%252Flinux%252Fl-cn-filelock%252F)
