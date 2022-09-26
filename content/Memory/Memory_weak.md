#  SideTable&weak底层原理

## 1. SideTable底层结构

> 来看看SideTable的定义：
>
> ```objc
> struct SideTable {
>  spinlock_t slock;
>  RefcountMap refcnts;
>  weak_table_t weak_table;
> }
> 
> ```

SideTable的定义很清晰，有三个成员:

> 1. **<font color='red'>spinlock_t </font> slock**: 自旋锁，保证操作 `SideTable` 时的线程安全。看前面的两大块 `weak_table_t` 和 `weak_entry_t` 的时候，看到它们所有的操作函数都没有提及加解锁的事情，如果你仔细观察的话会发现它们的函数名后面都有一个 `no_lock` 的小尾巴，正是用来提醒我们，它们的操作完全并没有涉及加锁。其实它们是把保证它们线程安全的任务交给了 `SideTable`，下面可以看到 `SideTable` 提供的函数都是线程安全的，而这都是由 `slock` 来完成的。
>
> 2. **<font color='red'>RefcountMap </font> refcnts**: 以 `DisguisedPtr<objc_object>` 为 `key`，以 `size_t` 为 `value` 的哈希表，用来存储对象的引用计数（仅在未使用 `isa` 优化或者 `isa` 优化情况下 `isa_t` 中保存的引用计数溢出时才会用到，这里涉及到 `isa_t` 里的 `uintptr_t has_sidetable_rc` 和 `uintptr_t extra_rc` 两个字段，以前只是单纯的看 `isa` 的结构，到这里终于被用到了，还有这时候终于知道 `rc` 其实是 `refcount`(引用计数) 的缩写）。作为哈希表，它使用的是平方探测法从哈希表中取值，而 `weak_table_t` 则是线性探测（开放寻址法）。
>
>    (仅在未开启isa优化或在isa优化情况下isa_t的引用计数溢出时才会用到，<font color="red">**未溢出时是放在isa_t下的extra_rc字段中**</font>)。
>
> 3. **<font color='red'>weak_table_t </font> weak_tabl**e： 存储对象弱引用的哈希表，是 `weak` 功能实现的核心数据结构。

> <img src="/Users/tangh/yuki/博客/文章仓库/YLNoteHub/content/Memory/image/Memory_weak-weak_SideTables.png" alt="img" style="zoom:80%;" />

> <img src="/Users/tangh/yuki/博客/文章仓库/YLNoteHub/content/Memory/image/Memory_weak-weak_table.png" alt="weak_table" style="zoom:125%;" />



* SideTables哈希数组 (个人理解为数组下标是通过向某一个哈希函数传入key得到，然后从数组中直接取值)

* weak-table：哈希表（也称散列表， 哈希表本质是一个数组，数组中的每一个元素成为一个箱子，箱子中存放的是键值对）

  两者关系如图：

**总结：**

​      底层数据模型：哈希表+数组的形式

（key: 弱引用对象的地址; value：一个存放weak指针的地址的数组）

## 2. weak实现原理

### 2.1 objc_initWeak  

`0bjc_initWeak(location,newObj)`

```C
id objc_initWeak(id *location, id newObj)
{
    if (!newObj) {
        *location = nil;
        return nil;
    }

    return storeWeak<DontHaveOld, DoHaveNew, DoCrashIfDeallocating>
        (location, (objc_object*)newObj);
}
复制代码
```

该方法的两个参数`location`和`newObj`。

> - **location** ：**`__weak指针`的地址**，存储指针的地址，这样便可以在最后将其指向的对象置为nil。
> - **newObj** ：所引用的对象。即例子中的obj 。

从上面的代码可以看出`objc_initWeak`方法只是一个深层次函数调用的入口，在该方法内部调用了`storeWeak `方法。下面我们来看下`storeWeak `方法的实现代码。

### **2.2 storeWeak**

```c
template <HaveOld haveOld, HaveNew haveNew, CrashIfDeallocating crashIfDeallocating> static id storeWeak(id *location, objc_object *newObj)
```

总结：

`storeWeak `方法的实现代码虽然有些长，但是并不难以理解。下面我们来分析下该方法的实现。

> 1. `storeWeak`方法实际上是接收了5个参数，分别是`haveOld、haveNew和crashIfDeallocating `，这三个参数都是以模板的方式传入的，是三个bool类型的参数。 分别表示weak指针之前是否指向了一个弱引用，weak指针是否需要指向一个新的引用，若果被弱引用的对象正在析构，此时再弱引用该对象是否应该crash。
> 2. 该方法维护了`oldTable `和`newTable`分别表示旧的引用弱表和新的弱引用表，它们都是`SideTable`的hash表。
> 3. 如果weak指针之前指向了一个弱引用，则会调用`weak_unregister_no_lock `方法将旧的weak指针地址移除。
> 4. 如果weak指针需要指向一个新的引用，则会调用`weak_register_no_lock `方法将新的weak指针地址添加到弱引用表中。
> 5. 调用`setWeaklyReferenced_nolock `方法修改weak新引用的对象的bit标志位



### 2.3 对象的dealloc 全过程

> 当对象的引用计数为0时，底层会调用`_objc_rootDealloc`方法对对象进行释放，而在`_objc_rootDealloc`方法里面会调用`rootDealloc`方法。如下是`rootDealloc`方法的代码实现。
>
> ```C
> inline void
> objc_object::rootDealloc()
> {
>     if (isTaggedPointer()) return;  // fixme necessary?
> 
>     if (fastpath(isa.nonpointer  &&  
>                  !isa.weakly_referenced  &&  
>                  !isa.has_assoc  &&  
>                  !isa.has_cxx_dtor  &&  
>                  !isa.has_sidetable_rc))
>     {
>         assert(!sidetable_present());
>         free(this);
>     } 
>     else {
>         object_dispose((id)this);
>     }
> }
> 复制代码
> ```
>
> > 1. 首先判断对象是否是`Tagged Pointer`，如果是则直接返回。
> > 2. 如果对象是采用了优化的isa计数方式，且同时满足对象没有被weak引用`!isa.weakly_referenced`、没有关联对象`!isa.has_assoc `、没有自定义的C++析构方法`!isa.has_cxx_dtor`、没有用到SideTable来引用计数`!isa.has_sidetable_rc`则直接快速释放。
> > 3. 如果不能满足2中的条件，则会调用`object_dispose `方法。



#### object_dispose

`object_dispose `方法很简单，主要是内部调用了`objc_destructInstance`方法。

```C
void *objc_destructInstance(id obj) 
{
    if (obj) {
        // Read all of the flags at once for performance.
        bool cxx = obj->hasCxxDtor();
        bool assoc = obj->hasAssociatedObjects();

        // This order is important.
        if (cxx) object_cxxDestruct(obj);
        if (assoc) _object_remove_assocations(obj);
        obj->clearDeallocating();
    }

    return obj;
}

```

上面这一段代码很清晰，如果有自定义的C++析构方法，则调用C++析构函数。如果有关联对象，则移除关联对象并将其自身从`Association Manager`的map中移除。调用`clearDeallocating `方法清除对象的相关引用。

#### clearDeallocating

```C
inline void 
objc_object::clearDeallocating()
{
    if (slowpath(!isa.nonpointer)) {
        // Slow path for raw pointer isa.
        sidetable_clearDeallocating();
    }
    else if (slowpath(isa.weakly_referenced  ||  isa.has_sidetable_rc)) {
        // Slow path for non-pointer isa with weak refs and/or side table data.
        clearDeallocating_slow();
    }

    assert(!sidetable_present());
}

```

`clearDeallocating`中有两个分支，先判断对象是否采用了优化`isa`引用计数，如果没有的话则需要清理对象存储在SideTable中的引用计数数据。如果对象采用了优化`isa`引用计数，则判断是否有使用SideTable的辅助引用计数(`isa.has_sidetable_rc`)或者有weak引用(`isa.weakly_referenced`)，符合这两种情况中一种的，调用`clearDeallocating_slow `方法。

#### clearDeallocating_slow（weak变量的清理工作从这里开始）

```C
NEVER_INLINE void
objc_object::clearDeallocating_slow()
{
    assert(isa.nonpointer  &&  (isa.weakly_referenced || isa.has_sidetable_rc));

    SideTable& table = SideTables()[this]; // 在全局的SideTables中，以this指针为key，找到对应的SideTable
    table.lock();
    if (isa.weakly_referenced) { // 如果obj被弱引用
        weak_clear_no_lock(&table.weak_table, (id)this); // 在SideTable的weak_table中对this进行清理工作
    }
    if (isa.has_sidetable_rc) { // 如果采用了SideTable做引用计数
        table.refcnts.erase(this); // 在SideTable的引用计数中移除this
    }
    table.unlock();
}

```

在这里我们关心的是`weak_clear_no_lock `方法。这里调用了weak_clear_no_lock来做weak_table的清理工作。

#### weak_clear_no_lock

```C
void 
weak_clear_no_lock(weak_table_t *weak_table, id referent_id) 
{
    objc_object *referent = (objc_object *)referent_id;

    weak_entry_t *entry = weak_entry_for_referent(weak_table, referent); // 找到referent在weak_table中对应的weak_entry_t
    if (entry == nil) {
        /// XXX shouldn't happen, but does with mismatched CF/objc
        //printf("XXX no entry for clear deallocating %p\n", referent);
        return;
    }

    // zero out references
    weak_referrer_t *referrers;
    size_t count;
    
    // 找出weak引用referent的weak 指针地址数组以及数组长度
    if (entry->out_of_line()) {
        referrers = entry->referrers;
        count = TABLE_SIZE(entry);
    } 
    else {
        referrers = entry->inline_referrers;
        count = WEAK_INLINE_COUNT;
    }
    
    for (size_t i = 0; i < count; ++i) {
        objc_object **referrer = referrers[i]; // 取出每个weak ptr的地址
        if (referrer) {
            if (*referrer == referent) { // 如果weak ptr确实weak引用了referent，则将weak ptr设置为nil，这也就是为什么weak 指针会自动设置为nil的原因
                *referrer = nil;
            }
            else if (*referrer) { // 如果所存储的weak ptr没有weak 引用referent，这可能是由于runtime代码的逻辑错误引起的，报错
                _objc_inform("__weak variable at %p holds %p instead of %p. "
                             "This is probably incorrect use of "
                             "objc_storeWeak() and objc_loadWeak(). "
                             "Break on objc_weak_error to debug.\n", 
                             referrer, (void*)*referrer, (void*)referent);
                objc_weak_error();
            }
        }
    }
    
    weak_entry_remove(weak_table, entry); // 由于referent要被释放了，因此referent的weak_entry_t也要移除出weak_table
}
```

附录：<font color='red'> 注意weak_table_t在移除每个weak指针时取了每个referrer的地址与当前对象比较，如果相等则证明是匹配的（weak ptr确实指向了referent）</font>

### 总结

- **1、weak的原理在于底层维护了一张weak_table_t结构的hash表，key是所指对象的地址，value是weak指针的地址数组。**
- **2、weak 关键字的作用是弱引用，所引用对象的计数器不会加1，并在引用对象被释放的时候自动被设置为 nil。**
- **3、对象释放时，调用`clearDeallocating`函数根据对象地址获取所有weak指针地址的数组，然后遍历这个数组把其中的数据设为nil，最后把这个entry从weak表中删除，最后清理对象的记录。**
- **4、文章中介绍了SideTable、weak_table_t、weak_entry_t这样三个结构，它们之间的关系如下图所示。**


