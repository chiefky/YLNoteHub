## _Block_object_assign 方法分析	

_Block_object_assign 方法支持的外部变量主要有以下几种（也代表Block 可捕获的外界变量的种类）：

```c++
// Runtime support functions used by compiler when generating copy/dispose helpers 
// Values for _Block_object_assign() and _Block_object_dispose() parameters 
enum { 
  // see function implementation for a more complete description of these fields and combinations 

  //普通对象，即没有其他的引用类型 
* BLOCK_FIELD_IS_OBJECT = 3, // id, NSObject, __attribute__((NSObject)), block, ... 

   // block类型作为变量 
* BLOCK_FIELD_IS_BLOCK = 7, // a block variable 

  // 经过__block修饰的变量 
* BLOCK_FIELD_IS_BYREF = 8, // the on stack structure holding the __block variable 

  // weak 弱引用变量 
* BLOCK_FIELD_IS_WEAK = 16, // declared __weak, only used in byref copy helpers 

  // 返回的调用对象 - 处理block_byref内部对象内存会加的一个额外标记，配合flags一起使用 
* BLOCK_BYREF_CALLER = 128, // called from __block (byref) copy/dispose support routines. 
};
```

_Block_object_assign 内部具体实现：
* 如果是普通对象(BLOCK_FIELD_IS_OBJECT)，则交给系统arc处理，并拷贝对象指针，即引用计数+1，所以外界变量不能释放
* 如果是block类型的变量(BLOCK_FIELD_IS_BLOCK)，则通过_Block_copy操作，将block从栈区拷贝到堆区
* 如果是 __block 修饰的变量(BLOCK_FIELD_IS_BYREF)，调用_Block_byref_copy函数 进行内存拷贝以及常规处理

<font color='orange'>Tip: 以上可理解为“block的三层copy”</font>

