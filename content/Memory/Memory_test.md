# 面试题

## 1. 一个NSObject对象占用多少内存空间？（问的是实际占用空间，不是系统分配空间）

答：一个NSObject对象的结构体中，只有isa一个成员，isa是指针类型。因此，在64位架构中，占用**8个字节**的内存。在这里，结构体中只有一个成员，所以isa的地址，就是结构体的地址，就是NSObject对象的地址。

> >##### *  一个NSObject对象占用多少内存？
> >
> >> 答：系统分配了16个字节给NSObject对象（通过malloc_size函数获得）
> >>  但NSObject对象内部只使用了8个字节的空间（64bit环境下，可以通过class_getInstanceSize函数获得）
> >>
> >> - class_getInstanceSize([NSObject class]);//创建一个实例对象至少需要多少内存空间   8字节
> >> - malloc_size((__bridge const void *)obj);//创建一个实例对象系统分配了多少内存空间   16字节
> >>   因为isa指针在64bit下是8个字节，而iOS手机端在64bit下系统分配的内存是16的整数倍，最少是16个字节
> >
> >##### *  如何获取NSObject对象的内存大小？
> >
> >> 获取NSObject对象的内存大小，需要用到以下几个函数：
> >>
> >> * **class_getInstanceSize** 实例对象实际占用的大小
> >>
> >> * **malloc_size** 系统分配的大小
> >>
> >> * **sizeOf** 一种操作符，计算类型的占用大小
> >>
> >>   其中，sizeof确切来说并不算做函数，它是一种操作符。
> >>
> >>   ```objective-c
> >>   NSObject *obj = [[NSObject alloc] init];
> >>   NSLog(@"class_getInstanceSize = %zd", class_getInstanceSize([NSObject class]));
> >>   NSLog(@"malloc_size = %zd", malloc_size((__bridge const void *)(obj)));     
> >>   NSLog(@"sizeOf = %zd", sizeof(obj));
> >>   ```
> >>
> >>   ```c++
> >>   输出：
> >>   class_getInstanceSize = 8
> >>   malloc_size = 16
> >>   sizeOf = 8
> >>   ```

## 2. 一个自定义类的对象占用多大的内存空间？

答：isa(8字节) + 各变量占用空间

## 3. weak 和 unowned

`weak` 关键字能将循环引用中的一个强引用替换为弱引用，以此来破解循环引用。

而还有另一个关键字 `unowned`，通过将强引用替换为无主引用，也能破解循环引用.

### 3.1 不过二者有什么区别呢？

弱引用对象可以为 `nil`，而无主引用对象不能，会发生运行时错误。

比如上面的例子我们使用了 `weak`，那么就需要额外使用 `guard let` 进行一步解包。而如果使用 `unowned`，就可以省略解包的一步：

```swift
someClosure = { [unowned self] in
    return self.a + self.b
}
```

`weak` 在底层添加了附加层，间接地把 `unowned` 引用包裹到了一个可选容器里面，虽然这样做会更加清晰，但是在性能方面带来了一些影响，所以 `unowned` 会更快一些。

但是无主引用有可能导致 crash，就是无主引用的对象为 `nil` 时，比如上面这个例子中，`anotherFunction` 我们会延迟 5s 调用 `someClosure`，但是如果 5s 内我们已经 pop 了这个 `viewController`，那么 `unowned self` 在调用时就会发现 `self` 已经被释放了，此时就会发生崩溃。

综上：

如果简单类比，使用 `weak` 的引用对象就类似于一个可选类型，使用时需要考虑解包；而使用 `unowned` 的引用对象就类似于已经进行强制解包了，不需要再解包，但是如果对象是 `nil`，那么就会直接 crash。

### 3.2 到底什么情况下可以使用 `unowned` 呢？

根据官方文档 Automatic Reference Counting 所说，无主引用在其他实例有相同或者更长的生命周期时使用。

## 4. iOS如何处理内存警告？

如何监控内存警告，以及处理 Jetsam 事件呢？

首先，内核会调起一个内核优先级最高（`95 /* MAXPRI_KERNEL */` 已经是内核能给线程分配的最高优先级了）的线程：

```
// 同样在 bsd/kern/kern_memorystatus.c 文件中
result = kernel_thread_start_priority(memorystatus_thread, NULL, 95 /* MAXPRI_KERNEL */, &jetsam_threads[i].thread);
```

这个线程会维护两个列表，一个是基于优先级的进程列表，另一个是每个进程消耗的内存页的列表。与此同时，它会监听内核 `pageout` 线程对整体内存使用情况的通知，在内存告急时向每个进程转发内存警告，也就是触发 `didReceiveMemoryWarning` 方法。

而杀掉应用，触发 OOM，主要是通过 `memorystatus_kill_on_VM_page_shortage`，有同步和异步两种方式。同步方式会立刻杀掉进程，先根据优先级，杀掉优先级低的进程；同一优先级再根据内存大小，杀掉内存占用大的进程。而异步方式只会标记当前进程，通过专门的内存管理线程去杀死。

### 

## other

一般来说做点题才能加深理解和巩固，所以这里从文章里简单提炼了一些，希望能帮到大家：

1. 什么是冯·诺依曼结构？
2. 什么是冯·诺依曼结构的瓶颈，以及如何突破瓶颈？
3. 存储器分哪两类，分别有什么特点？
4. 为什么使用缓存能提高效率？
5. 什么是物理寻址？什么是虚拟寻址？
6. 虚拟地址翻译过程由谁负责？具体流程是怎样的？
7. 虚拟内存有哪些意义？
8. 什么是内存交换机制？
9. 内存分页有什么意义？
10. iOS 的内存机制有什么特点？
11. clean memory、dirty memory、compressed memory 分别是什么？
12. 引起循环引用的本质原因是什么？
13. weak 和 unowned 的区别是什么？
14. 列举一些不会导致循环引用的闭包场景。
15. 什么是 OOM 崩溃？
16. 检测 OOM 崩溃有哪些常见方法？
17. OOM 崩溃有哪些常见原因？
