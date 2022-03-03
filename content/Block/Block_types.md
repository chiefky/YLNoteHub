# Block的三种类型

### 三种类型特点：

(堆栈是在运行时分配空间，全局是在编译期就能确定)

* 全局block
  * 位于全局区
  * 在block内部不使用外部变量 or **只使用**全局变量或者静态变量
* 堆区block
  * 位于堆区
  * 前提：在block内部可以使用外部变量或OC属性，并且将block赋值给strong或copy修饰的变量
* 栈区block
  * 位于栈区
  * 前提：在block内部可以使用外部变量或OC属性，并且不对block赋值或者只能赋值给weak修饰的变量

举例说明：

<img src="/Users/tangh/yuki/博客/文章仓库/YLNoteHub/content/images/block_声明.png" alt="声明" style="zoom:80%;" />

<img src="/Users/tangh/yuki/博客/文章仓库/YLNoteHub/content/images/block_类型_global.png" alt="全局区block" style="zoom:80%;" />

<img src="/Users/tangh/yuki/博客/文章仓库/YLNoteHub/content/images/block_类型_malloc_statck.png" alt="堆区和栈区" style="zoom:80%;" />



### block作用域问题

```objective-c
#pragma mark - 作用域问题
- (void)testStackBlock_copy {
    int num = 8;
    void(^__weak weakBlock)(void) = nil;
    {
        void(^__weak weakBlock1)(void) = ^{
            NSLog(@"num = %d",num);
        };
        weakBlock = weakBlock1; //[weakBlock1 copy];
        /**
         测试：
         1." weakBlock = weakBlock1; "---正常执行
         两个block都是stack block，都存储在栈区；栈区空间是方法体空间，中间的'{}'表示匿名作用域，（C语言中介绍）栈存储期间匿名作用域的变量超出匿名作用域不一定立即释放；
         2. "weakBlock = [weakBlock1 copy];"---崩溃
         经过copy后 weakBlock指向的是堆区的空间,此空间在给weakBlock赋值前被ARC插入的release释放掉，所以weakBlock为null;
         超出"}"调用weakBlock(),相当于访问了野指针所以会崩溃。
         */

    }
    weakBlock();
}

```

