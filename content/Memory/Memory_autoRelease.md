# 1. Autoreleasepool

## 1.1 什么是Autoreleasepool?

自动释放池是 `Objective-C` 开发中的一种自动内存回收管理的机制，为了替代开发人员手动管理内存，实质上是使用编译器在适当的位置插入`release`、`autorelease`等内存释放操作。当对象调用 `autorelease `方法后会被放到自动释放池中延迟释放时机，当缓存池需要清除`dealloc`时，会向这些 `Autoreleased `对象做 `release` 释放操作。

## 1.2 对象什么时候释放

以下A、B两种情况person分别什么时候释放(dealloc触发时机)？

```objective-c
// A:
__weak id temp = nil;
{
    Person *person = [[Person alloc] init];
    temp = person;
}
// 插入的release操作
 // call void @llvm.objc.storeStrong(i8** %19, i8* null) #3  ----retainCount == 0，dealloc
// 插入NSLog 传入weak变量操作
 // %20 = call i8* @llvm.objc.loadWeakRetained(i8** %6) #3
NSLog(@"temp = %@",temp);
// 插入 weak release和destory
 // call void @llvm.objc.release(i8* %20) #3, !clang.imprecise_release !10
 // call void @llvm.objc.destroyWeak(i8** %6) #3
 // call void @llvm.objc.autoreleasePoolPop(i8* %10)

```

答案：

> 先打印： [person dealloc]
>
> 后打印： temp = （null）
>
> retainCount 减到0，立即释放，触发dealloc

```objective-c
// B:

+(instancetype)defaultPerson
{
    return [[Person alloc] init];
}

__weak id temp = nil;
{
    Person *person = [Person defaultPerson]; 
   // %16 = notail call i8* @llvm.objc.retainAutoreleasedReturnValue(i8* %14) #3 ----retainCount == 2，retainAutoreleasedReturnValue中调了objc_retain(obj);

    temp = person;
}
 // call void @llvm.objc.storeStrong(i8** %21, i8* null) #3 ----retainCount == 1，延迟释放
 // %22 = call i8* @llvm.objc.loadWeakRetained(i8** %6) #3
//  invoke void (i8*, ...) @NSLog(i8* bitcast (%struct.__NSConstantString_tag* @_unnamed_cfstring_.35 to i8*), i8* %22) to label %23 unwind label %24
NSLog(@"temp = %@",temp);
//   call void @llvm.objc.release(i8* %22) #3, !clang.imprecise_release !10
//  call void @llvm.objc.destroyWeak(i8** %6) #3
//  call void @llvm.objc.autoreleasePoolPop(i8* %10)

```

答案：

> 先打印： temp = **<Person: 0x10133d460>**
>
> 后打印： [person dealloc]
>
> retainCount减到1，对象延迟释放，等到autoreleasepool pop时，向pool内的对象发realse消息减到0才释放；



## 1.3 AutoreleasePool的使用

**初始化or创建**

MRC下：

```objective-c
//1.生成一个NSAutoreleasePool对象
NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
//2.创建对象
id object = [[NSObject alloc] init];
//3.对象调用autorelease方法
[object autorelease];
//1.废弃NSAutoreleasePool对象，会对释放池中的object发送release消息
[pool drain];

```

ARC:

```objective-c
@autoreleasepool {
    //LLVM会在内部插入autorelease方法
}

```

### TODO: 面试题：

以下代码有什么区别吗？

```objc
@implementation YLAnimal
- (void)dealloc {
    NSLog(@"%s",__func__);
}
+ (instancetype)animal {
  YLAnimal *ani = [YLAnimal alloc];
  return ani;
}
@end
  
// A
@autoreleasepool {
  __weak id temp = nil;
  {
    YLAnimal *animal = [[YLAnimal alloc] init];
    temp = animal;
  }
  NSLog(@"temp = %@",temp);
}

// B
@autoreleasepool {
  __weak id temp = nil;
  {
      YLAnimal *animal = [YLAnimal animal];
      temp = animal;
  }
  NSLog(@"temp = %@",temp);
}

```

答：`alloc`方法初始化 的animal不会加入到autoreleasepool中，超出作用域后，retaincount减为0，立即释放；

`anima`方法初始化的animal会加入到autoreleasepool中，超出作用域后，延迟到autoreleasepool pop，才会释放;

> 引申：
>
> 加入到autoreleasepool的对象一定会释放吗？
>
> 答：不一定，autoreleasepool销毁时只是对其内部对象发送release消息，引用计数减1，此时若对象的引用计数减完后不为0，就不会被释放，也就是所谓的内存泄漏。
>
> 引发内存泄漏的情况有哪些？
>
> * 循环引用：
>   * block
>   * A --> B --> A
>   * A --> B --> C --> A
>   * NSObserver center (<font color="red">待补充</font>) 
> * NSObserver center (<font color="red">待补充</font>) 
> * 

## 1.4  ARC下`@autoreleasepool {} `被编译成了什么

使用clang命令将main.m编译成main.cpp，查看相关代码，如下：

```cpp
//编译前：
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
    }
    
    return 0;
}

//编译后：
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

    }

    return 0;
}
```

可见，编译后把`@autoreleasePool`转换成一个`__AtAutoreleasePool `类型的局部私有变量`__AtAutoreleasePool __autoreleasepool`;然后查看一下`__AtAutoreleasePool`是什么？

对应main.cpp文件，main函数前面有`__AtAutoreleasePool`源码结构，如下：

```cpp
struct __AtAutoreleasePool {
  __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
  ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
  void * atautoreleasepoolobj;
};
```

`__AtAutoreleasePool`是结构体类型，其内包含：

* 构造函数`__AtAutoreleasePool()`
* 析构函数`~__AtAutoreleasePool(atautoreleasepoolobj)`。
* atautoreleasepoolobj (就是我们常说的`POOL_BOUNDARY`桩对象，析构时，以此对象为参照物向此对象之后插入的对象全部发送release消息)

关于`@autoreleasepool`就这些，接下来是`objc_autoreleasePoolPush()` 、`objc_autoreleasePoolPop(obj)`的实现，即关于`AutoreleasepoolPage`的探索；

```cpp
void *
objc_autoreleasePoolPush(void)
{
    return AutoreleasePoolPage::push();
}

NEVER_INLINE
void
objc_autoreleasePoolPop(void *ctxt)
{
    AutoreleasePoolPage::pop(ctxt);
}

```

# 2. AutoreleasepoolPage

## 2.1 内存结构

### AutoreleasepoolPage在内存中以双向链表形式存在，如图：

![双向链表](/Users/tangh/yuki/iOS练习Demos/YLNote/YLNote/latest/个人笔记/文章/images/内存管理-AutoreleasePoolPage_0.png)

### 单个page的内存形态



<img src="/Users/tangh/yuki/iOS练习Demos/YLNote/YLNote/latest/个人笔记/文章/images/内存管理-autoreleasepage_1.png" alt="AutoreleasepoolPage" style="zoom:80%;" />

`next` 指向了下一个为空的内存地址，如果 `next` 指向的地址加入一个 `object`，它就会如下图所示**移动到下一个为空的内存地址中**：

> 注意：各变量在内存中的位置；

## 2.2  AutoreleasePoolPage部分关键源码

```cpp
class AutoreleasePoolPage : private AutoreleasePoolPageData
{
	friend struct thread_data_t;

public:
	static size_t const SIZE =
#if PROTECT_AUTORELEASEPOOL
		PAGE_MAX_SIZE;  // must be multiple of vm page size
#else
		PAGE_MIN_SIZE;  // size and alignment, power of 2
#endif
    
private:
	static pthread_key_t const key = AUTORELEASE_POOL_KEY;
	static uint8_t const SCRIBBLE = 0xA3;  // 0xA3A3A3A3 after releasing
	static size_t const COUNT = SIZE / sizeof(id);
    static size_t const MAX_FAULTS = 2;

    // EMPTY_POOL_PLACEHOLDER is stored in TLS when exactly one pool is 
    // pushed and it has never contained any objects. This saves memory 
    // when the top level (i.e. libdispatch) pushes and pops pools but 
    // never uses them.
#   define EMPTY_POOL_PLACEHOLDER ((id*)1)

#   define POOL_BOUNDARY nil

	AutoreleasePoolPage(AutoreleasePoolPage *newParent) :
		AutoreleasePoolPageData(begin(),
								objc_thread_self(),
								newParent,
								newParent ? 1+newParent->depth : 0,
								newParent ? newParent->hiwat : 0){ }

    ~AutoreleasePoolPage(){}


  id *add(id obj){}
  static inline void *push() ;
  static inline void pop(void *token);
  static inline AutoreleasePoolPage *hotPage();
  static inline void setHotPage(AutoreleasePoolPage *page) 
  static inline AutoreleasePoolPage *coldPage() 
    .
    .
    .

}


```

```cpp
class AutoreleasePoolPage;
struct AutoreleasePoolPageData
{
#if SUPPORT_AUTORELEASEPOOL_DEDUP_PTRS
    struct AutoreleasePoolEntry {
        uintptr_t ptr: 48;
        uintptr_t count: 16;

        static const uintptr_t maxCount = 65535; // 2^16 - 1
    };
    static_assert((AutoreleasePoolEntry){ .ptr = MACH_VM_MAX_ADDRESS }.ptr == MACH_VM_MAX_ADDRESS, "MACH_VM_MAX_ADDRESS doesn't fit into AutoreleasePoolEntry::ptr!");
#endif

	magic_t const magic; //用来校验 `AutoreleasePoolPage`的结构是否完整；16
	__unsafe_unretained id *next; //指向最新添加的 `autoreleased` 对象的下一个位置，初始化时指向  `begin()` ；8
	pthread_t const thread;  //指向当前线程；8
	AutoreleasePoolPage * const parent;    //指向父结点，第一个结点的 `parent` 值为 `nil` ；8
	AutoreleasePoolPage *child;     //指向子结点，最后一个结点的 `child` 值为 `nil` ；8
	uint32_t const depth;  //代表深度，从 `0` 开始，往后递增 `1`；4
	uint32_t hiwat; //代表 `high water mark` ; 4

	AutoreleasePoolPageData(__unsafe_unretained id* _next, pthread_t _thread, AutoreleasePoolPage* _parent, uint32_t _depth, uint32_t _hiwat)
		: magic(), next(_next), thread(_thread),
		  parent(_parent), child(nil),
		  depth(_depth), hiwat(_hiwat)
	{
	}
};
```

**通过源码直观可见：**

1. `AutoreleasePoolPage`是继承自`AutoreleasePoolPageData`。
2. `AutoreleasePoolPageData`有一个parent、一个child变量，分别是一个`AutoreleasePoolPage`指针。
3. 每一个`AutoreleasePoolPageData`都对应一个pthread_t线程
4. `AutoreleasePoolPageData`结构体的大小为56字节

### 2.2.1 AutoreleasePoolPage::push()

```cpp
static inline void *push() 
    {
        id *dest;
          //判断是否已经初始化AutoreleasePoolPage，slowpath()为小概率发生事件
        if (slowpath(DebugPoolAllocation)) {
            // Each autorelease pool starts on a new pool page.
            dest = autoreleaseNewPage(POOL_BOUNDARY); 
        } else {
            dest = autoreleaseFast(POOL_BOUNDARY);
        }
        // ⬆️ 这里的POOL_BOUNDARY就是我们常说的哨兵对象
        ASSERT(dest == EMPTY_POOL_PLACEHOLDER || *dest == POOL_BOUNDARY);
        return dest;
    }

```

##### 关于"autoreleaseFast"核心方法的概括：

* autoreleaseFullPage() ; // hotpage已满: 如果子页面存在，则将页面替换为子页面;不存在，则创建新页面

* autoreleaseNoPage()；// hotpage不存在;

  > 第一步：先判断“是否有空占位符”，然后该标记标记，该报错报错，该返回empty pool placeholder返回empty pool placeholder；
  >
  > 第二步：确保占位符状态ok，且已经有了boundary，然后初始化一个page，并设置为hotPage；
  >
  > 第三步：page和占位符都就绪，先插入boundry对象，再add(obj)

* add(obj）；// hotpage存在，且未满：将当前对象加入pool中；(疑问❓ 这里返回的是谁的指针)



#### autoreleaseNewPage()方法--小概率发生事件

> ```cpp
> static __attribute__((noinline))
> id *autoreleaseNewPage(id obj)
> {
>  //获取当前hotpage
>  AutoreleasePoolPage *page = hotPage();
>  //判断当前页是否存在，如果存在，则压栈对象
>  if (page) return autoreleaseFullPage(obj, page);
>  //如果不存在，则创建page
>  else return autoreleaseNoPage(obj);
> }
> 
> ```

#### autoreleaseFast()方法--大概率发生事件

> ```cpp
>  static inline id *autoreleaseFast(id obj)
>  {
>      AutoreleasePoolPage *page = hotPage();
>      if (page && !page->full()) {
>          return page->add(obj);
>      } else if (page) {
>          return autoreleaseFullPage(obj, page);
>      } else {
>          return autoreleaseNoPage(obj);
>      }
>  }
> 
> ```

###### autoreleaseFullPage() 方法

```cpp
static __attribute__((noinline))
id *autoreleaseFullPage(id obj, AutoreleasePoolPage *page)
{
    // The hot page is full. 
    // Step to the next non-full page, adding a new page if necessary.
    // Then add the object to that page.
    ASSERT(page == hotPage());
    ASSERT(page->full()  ||  DebugPoolAllocation);
    
    //遍历循环查找page是否已满
    do {
        //如果子页面存在，则将页面替换为子页面
        if (page->child) page = page->child;
        //如果子页面不存在，则创建新页面
        else page = new AutoreleasePoolPage(page);
    } while (page->full());

    //设置为当前hotPage
    setHotPage(page);
    //对象压栈
    return page->add(obj);
}
```

###### autoreleaseNoPage()方法

```cpp
static __attribute__((noinline))
id *autoreleaseNoPage(id obj)
{
    // "No page" could mean no pool has been pushed
    // or an empty placeholder pool has been pushed and has no contents yet
    ASSERT(!hotPage());

    bool pushExtraBoundary = false;
    //判断是否是空占位符，如果是，则压栈哨兵标识符置为YES
    if (haveEmptyPoolPlaceholder()) {
        pushExtraBoundary = true;
    }
    //如果对象不是哨兵对象，且没有Pool，则报错
    else if (obj != POOL_BOUNDARY  &&  DebugMissingPools) {
        _objc_inform("MISSING POOLS: (%p) Object %p of class %s "
                     "autoreleased with no pool in place - "
                     "just leaking - break on "
                     "objc_autoreleaseNoPool() to debug", 
                     objc_thread_self(), (void*)obj, object_getClassName(obj));
        objc_autoreleaseNoPool(obj);
        return nil;
    }
    //如果对象是哨兵对象，且没有申请自动释放池内存，则设置一个空占位符存储在tls中，其目的是为了节省内存
    else if (obj == POOL_BOUNDARY  &&  !DebugPoolAllocation) {
        // We are pushing a pool with no pool in place,
        // and alloc-per-pool debugging was not requested.
        // Install and return the empty pool placeholder.
        return setEmptyPoolPlaceholder();//设置空的占位符
    }

    // We are pushing an object or a non-placeholder'd pool.

    // Install the first page.
    //初始化第一页
    AutoreleasePoolPage *page = new AutoreleasePoolPage(nil);
    //设置page为当前hotpage
    setHotPage(page);
    
    // Push a boundary on behalf of the previously-placeholder'd pool.
    //压栈哨兵的标识符为YES，则压栈哨兵对象
    if (pushExtraBoundary) {
        //压栈哨兵
        page->add(POOL_BOUNDARY);
    }
    
    // Push the requested object or pool.
    //压栈对象
    return page->add(obj);
}
```

###### add(obj)方法

```cpp
id *add(id obj)
{
    assert(!full());
    unprotect();
    id *ret = next;  // faster than `return next-1` because of aliasing
    *next++ = obj;
    protect();
    return ret;
}

```

### 2.2.2 AutoreleasePoolPage::pop()

<img src="/Users/tangh/yuki/iOS练习Demos/YLNote/YLNote/latest/个人笔记/文章/images/内存管理-autoreleasepage_pop_0.png" alt="pop-0" style="zoom:80%;" />

<img src="/Users/tangh/yuki/iOS练习Demos/YLNote/YLNote/latest/个人笔记/文章/images/内存管理-autoreleasepage_pop_1.png" alt="pop-1" style="zoom:80%;" />

<img src="/Users/tangh/yuki/iOS练习Demos/YLNote/YLNote/latest/个人笔记/文章/images/内存管理-autoreleasepage_pop_2.png" alt="pop-2" style="zoom:80%;" />

##### 其中核心三步：

* pageForPointer(token); 根据传入的哨兵对象的地址，取得哨兵对象所在的page地址；
* releaseUntil(stop); 拿hotPage逐一减1，直到page的next指向boundary；
* page->kill(); 方法删除双向链表中的每一个page

#### pageForPointer( ) 方法 ---- 根据哨兵对象找到page

> ```cpp
> static AutoreleasePoolPage *pageForPointer(uintptr_t p) 
> {
>  AutoreleasePoolPage *result;
>  uintptr_t offset = p % SIZE;
> 
>  ASSERT(offset >= sizeof(AutoreleasePoolPage));
> 
>  result = (AutoreleasePoolPage *)(p - offset);
>  result->fastcheck();
> 
>  return result;
> }
> ```
>
> 解读：
>
> 1. 哨兵对象地址与页面size(4096)取模--->哨兵对象的偏移量
> 2. 根据哨兵对象的地址减偏移量便可以得到根page的首地址--->哨兵对象所在page页

#### releaseUntil(stop)方法 ---- 释放对象

> ```cpp
> void releaseUntil(id *stop) 
> {
>  // Not recursive: we don't want to blow out the stack 
>  // if a thread accumulates a stupendous amount of garbage
> 
>  // 释放AutoreleasePoolPage中的对象，直到next指向stop
>  while (this->next != stop) {
>      // Restart from hotPage() every time, in case -release 
>      // autoreleased more objects
>      // hotPage可以理解为当前正在使用的page
>      AutoreleasePoolPage *page = hotPage();
> 
>      // fixme I think this `while` can be `if`, but I can't prove it
>      while (page->empty()) {
>          page = page->parent;
>          setHotPage(page);
>      }
> 
>      page->unprotect();
>      // obj = page->next; page->next--;
>   id obj = *--page->next;
>      memset((void*)page->next, SCRIBBLE, sizeof(*page->next));
>      page->protect();
> 
>      // POOL_BOUNDARY为nil，是哨兵对象
>      if (obj != POOL_BOUNDARY) {
>       // 释放obj对象
>          objc_release(obj);
>      }
>  }
> 
>  // 重新设置hotPage
>  setHotPage(this);
> 
> #if DEBUG
>  // we expect any children to be completely empty
> for (AutoreleasePoolPage *page = child; page; page = page->child) {
>   assert(page->empty());
>  }
> #endif
> }
> ```

#### kill()方法 ---- 删除双向链表中的每一个page

> ```cpp
> void kill() 
> {
>  // Not recursive: we don't want to blow out the stack 
>  // if a thread accumulates a stupendous amount of garbage
>  AutoreleasePoolPage *page = this;
>  // 找到链表最末尾的page
>  while (page->child) page = page->child;
> 
>  AutoreleasePoolPage *deathptr;
>  // 循环删除每一个page
>  do {
>      deathptr = page;
>      page = page->parent;
>      if (page) {
>          page->unprotect();
>          page->child = nil;
>          page->protect();
>      }
>      delete deathptr;
>  } while (deathptr != this);
> }
> ```

**通过源码方法实现解读的：**

1. `AutoreleasePoolPage`每个对象会开辟4096字节内存（虚拟内存一页的大小）

2. 除了上面`AutoreleasePoolPageData`的实例变量（结构体的大小）所占空间，剩下的空间全部用来储存`autorelease`对象的地址

   > （注意，`AutoreleasePoolPage`的第一页会包含哨兵对象，哨兵对象占位8字节，现每个加入的对象为8字节，那么第一页最多可以存储504个对象，从第二页开始最多可以存储505个对象。）

3. `AutoreleasepoolPage `通过压栈的方式来存储每个`autorelease`的对象(<font color=red>从低地址到高地址</font>)。

4. `next`指针作为游标指向栈顶最新`add`进来的`autorelease`对象的下一个位置,

   > 当 `next`指针指向`begin`时，表示 `AutoreleasePoolPage `为空；
   >
   > 当 `next`指针指向`end`时，表示 `AutoreleasePoolPage` 已满;

5. 当向`AutoreleasePool`中新增`autorelease`对象时，底层`autoreleaseFast`函数内会：

   * 当前`AutoreleasePoolPage`没满：直接将该对象插入此page中
   * 当前`AutoreleasePoolPage`满了：新建一个`AutoreleasePoolPage`对象(作为child，自然原来的page变成了parent)，并设置`child`为`hotPage`，`parent`为`coldPage`，将该对象插入hotPage中；
   * 当前`AutoreleasePoolPage`为空：新建一个`AutoreleasePoolPage`对象，设置为hotPage，再将该对象插入hotPage中；

6. 当销毁Autoreleasepool时,Autoreleasepool::pop函数内会：

   * 根据哨兵对象获取目标page地址
   * 通过root page拿到hotpage(),从hotPage的next指针开始向boundary之后的所有对象发送release消息
   * 所有对象发完release消息之后，调用kill方法删除空page （memory: delete empty children）；

# 3. AutoreleasePool的嵌套

查看两个嵌套的AutoreleasePool在内存中的结构：

源码：

```objective-c
 @autoreleasepool {
        for (int i = 0; i<5; i++) {
            YLAnimal *tmpAni = [YLAnimal animal2];
        }
        @autoreleasepool {
            for (int i = 0; i<10; i++) {
                YLDog *tmpAni = [YLDog doggy2];
            }
            _objc_autoreleasePoolPrint();
        }
        
    }
```

打印结果：

<img src="/Users/tangh/yuki/iOS练习Demos/YLNote/YLNote/latest/个人笔记/文章/images/内存管理-autoreleasepool_嵌套.png" alt="pool嵌套" style="zoom:80%;" />

其内部对象释放过程：

与单一autoreleasepool没有区别，主要是明白一点：**任何释放池在释放时都是从内侧pool开始释放，不存在一个直接从外层pool开始的情况；**

以上面结构为例，一定是先释放boundary 2 所在的pool；然后才能轮到 boundary 1 的pool；当释放到boundary 1 pool时，bound 2pool已经被销毁了，所以最终释放的都是一个单层的pool；

总结：

* 一个pool 有且只有一个boundary，但是可以有N个page
* 一个page内可能有1个、0个、多个boundary



# 4. +animal1、+animal2初始化的对象会自动加入autoreleasepool吗？

<img src="/Users/tangh/yuki/iOS练习Demos/YLNote/YLNote/latest/个人笔记/文章/images/内存管理-autorelease_汇编code.png" alt="是否自动加入pool" style="zoom:100%;" />

<img src="/Users/tangh/yuki/iOS练习Demos/YLNote/YLNote/latest/个人笔记/文章/images/内存管理-autoreleasepool_code_0.png" alt="是否自动加入pool" style="zoom:100%;" />

<img src="/Users/tangh/yuki/iOS练习Demos/YLNote/YLNote/latest/个人笔记/文章/images/内存管理-autoreleasepool_code_1.png" alt="是否自动加入pool" style="zoom:100%;" />

<img src="/Users/tangh/yuki/iOS练习Demos/YLNote/YLNote/latest/个人笔记/文章/images/内存管理-autoreleasepool_code_2.png" alt="是否自动加入pool" style="zoom:100%;" />

<img src="/Users/tangh/yuki/iOS练习Demos/YLNote/YLNote/latest/个人笔记/文章/images/内存管理-autoreleasepool_code_3.png" alt="是否自动加入pool" style="zoom:100%;" />

答：

* animal1初始化：经实验验证只有第一个对象加入了pool,之后初始化的对象都被标记为Optimized，直接return obj，并没有调objc_autorelease(obj)，后一步`objc_retainAutoreleasedReturnValue`方法中也没有对齐retain;
* animal2初始化：经实验验证所有对象都加入了pool，所有对象都没有被标记为优化对象；

> 相关面试题：
>
> 以下代码有没有什么问题?如果有?如何解决?
>
> ```objective-c
> for (int i = 0; i < 1000000; i++) {	
> 	NSString *str = [NSString stringWithFormat:@"hello -%04d", i];
> 	str = [str stringByAppendingString:@" - world"]; 
> }
> ```
>
> 问题：`stringWithFormat`将所有的对象的释放权都交给了RunLoop (主线程的)的释放池，而RunLoop的释放池会等待这个事件处理之后才会释放，由此延迟释放造成了内存堆积，使得内存出现峰值,如果当前手机内存使用率过高，可能会造成崩溃；
>
> 解决：
>
> ```objective-c
> for (int i = 0; i < 1000000; i++) {	
>      @autoreleasepool {
>          NSString *str = [NSString stringWithFormat:@"hello -%04d", i];
>          str = [str stringByAppendingString:@" - world"];
>      }
> }
> ```
>
> 这样每一次循环的结束时都会释放一次内存，因而这个循环全部执行完成时也几乎不消耗内存。
>
> 总结：
>
> * 主线程的RunLoop是默认开启的， 每一次消息循环开始的时候会先创建自动释放池，这次循环结束前,会释放自动释放池，然后RunLoop等待下次事件源。此Runloop类似一个全局Runloop，但其实并不是，要理解他是每一个迭代周期开始时重新初始化，结束时销毁掉；
> * 开发者可以在任何执行的地方创建释放池，也就是局部的释放池，这时的释放池类似于代码块 当释放池结束的时候会自动释放。
> * `stringWithFormat:`本质上等价于 alloc + initWithFormat: + autorelease （出自：https://www.cnblogs.com/Mike-zh/p/4445171.html）