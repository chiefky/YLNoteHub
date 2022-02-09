# 1. 关键字

## 1. define 和 const常量区别?



●define是宏定义，程序在预处理阶段将用define定义的内容进行了替换。因此程序运行时，常量表中并没有用define定义的常量，系统不为它分配内存。const定义的常量，在程序运行时在常量表中，系统为它分配内存。

●define定义的常量，预处理时只是直接进行了替换。所以编译时不能进行数据类型检验。
 const定义的常量，在编译时进行严格的类型检验，可以避免出错。

●const 常量有数据类型，而宏常量没有数据类型。编译器可以对前者进行类型安全检查。而对后者只进行字符替换，没有类型安全检查，并且在字符替换可能会产生意料不到的错误。

●define可以定义一些简单的函数，const不可以

●有些集成化的调试工具可以对const 常量进行调试，但是不能对宏常量进行调试

## 2. volatile关键字的作用及用途

`volatile`关键字的目的是防止编译器对变量访问做任何优化，因为这些变量可能会以编译器无法确定的方式被修改。

声明为`volatile`的变量不会被优化，因为它们的值随时可能被当前代码范围之外的代码修改。系统总是从内存读取变量的当前值，而不会使用寄存器中的值，即使上条指令刚操作过此数据。(`volatile`的影响远不止是否使用寄存器值这么简单)

### 应用场景

当变量的值可能发生意外变化时，应该将其声明为volatile。实际上，只有三种情况：

1. 内存映射外围设备寄存器
2. 由中断处理程序修改的全局变量
3. 被多个线程访问的共享变量

第一种情况，外围设备寄存器的值随时可能被外部改变，显然超出了代码的范围。第二种情况，中断处理程序的执行模式不同于普通程序，当中断到来时，当前线程挂起，执行中断处理程序，之后恢复代码的执行。可以认为，中断处理程序与当前程序是并行的，独立于正常代码执行序列之外。第三种情况比较常见，就是一般的并发编程。

## 3. static 、extern、const

```objc
// .m
NSInteger normal_global_var = 10;  // 生命周期：程序期间，作用域：全局
static NSInteger static_global_var = 10; // 生命周期：程序期间，作用域：声明文件内
```



### 3.1 static

在全局变量前加static, 全局变量就被定义成为一个全局静态变量（全局变量和静态全局变量的生命周期是一样的, 都是在堆中的静态区, 在整个工程执行期间内一直存在) 而静态全局变量则限制了其作用域, 即只在定义该变量的源文件内有效, 在同一源程序的其它源文件中不能使用它。

### 3.2 extern

**这个单词翻译过来是"外面的, 外部的"。 顾名思义, 它的作用是声明外部全局变量。这里需要特别注意 extern 只能声明, 不能用于实现。** 当使用 extern 来声明变量时, 其会先在编译单元内部进行查找, 如果没有则继续到外部进行查找, 如果缺少实现并且使用到了此数据时会导致编译不通过。

**用法:**

- 使用其来声明供外部使用。

最常用也是最常见的实现一般是, .h 用 extern 修饰可供外部使用, .m 实现.

🌰：

```objective-c
// .h
extern NSString* const JSDLoginManagerDidLoginNotification;
@interface JSDLoginManagerVC : ViewController
@end
```

```objc
// .m
NSString * const JSDLoginManagerDidLoginNotification = @"JSDLoginManagerDidLoginNotification";
@implementation JSDCrashVC
```

注意： (extern只能用来声明非静态全局变量，声明静态全局变量编译报错'Static declaration of 'static_global_var' follows non-static declaration')；

### 3.3 const

#### const与宏的区别:

- `const简介`:之前常用的字符串常量，一般是抽成宏，但是苹果不推荐我们抽成宏，推荐我们使用const常量。

  - `编译时刻`:宏是预编译（编译之前处理），const是编译阶段。
  - `编译检查`:宏不做检查，不会报编译错误，只是替换，const会编译检查，会报编译错误。
  - `宏的好处`:宏能定义一些函数，方法。 const不能。
  - `宏的坏处`:使用大量宏，容易造成编译时间久，每次都需要重新替换。

  注意:很多Blog都说使用宏，会消耗很多内存，我这验证并不会生成很多内存，宏定义的是常量，常量都放在常量区，只会生成一份内存。

#### const作用：限制类型

- 1.const仅仅用来修饰右边的变量（基本数据变量p，指针变量*p）
- 2.被const修饰的变量是只读的。

```objective-c
- (void)viewDidLoad {
    [super viewDidLoad];
    
    // 定义变量
    int a = 1;
    
    // 允许修改值
    a = 20;
    
    // const两种用法
    // const:修饰基本变量p
    // 这两种写法是一样的，const只修饰右边的基本变量b
    const int b = 20; // b:只读变量
    int const b = 20; // b:只读变量
    
    // 不允许修改值
    b = 1;
    
    // const:修饰指针变量*p，带*的变量，就是指针变量.
    // 定义一个指向int类型的指针变量，指向a的地址
    int *p = &a;
    
    int c = 10;
    
    p = &c;
    
    // 允许修改p指向的地址，
    // 允许修改p访问内存空间的值
    *p = 20;
    
    // const修饰指针变量访问的内存空间，修饰的是右边*p1，
    // 两种方式一样
    const int *p1; // *p1：常量 p1:变量
    int const *p1; // *p1：常量 p1:变量
    
    // const修饰指针变量p1
    int * const p1; // *p1:变量 p1:常量 【补充：p1是指针类型，*p1是p1指针指向的整型数int类型】

    
    // 第一个const修饰*p1 第二个const修饰 p1
    // 两种方式一样
    const int * const p1; // *p1：常量 p1：常量
    
    int const * const p1;  // *p1：常量 p1：常量
    
    
}
```

### const开发中使用场景:

- 1.需求1:提供一个方法，这个方法的参数是地址，里面只能通过地址读取值,不能通过地址修改值
- 2.需求2:提供一个方法，这个方法的参数是地址，里面不能修改参数的地址。



```objectivec
@implementation ViewController

// const放*前面约束参数，表示*a只读
// 只能修改地址a,不能通过a修改访问的内存空间
- (void)test:(const int * )a
{
//    *a = 20;
}

// const放*后面约束参数，表示a只读
// 不能修改a的地址，只能修改a访问的值
- (void)test1:(int * const)a
{
    int b;
    // 会报错
    a = &b;
    
    *a = 2;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    
    int a = 10;
    
    // 需求1:提供一个方法，这个方法的参数是地址，里面只能通过地址读取值,不能通过地址修改值。
    
    // 这时候就需要使用const，约束方法的参数只读.
    [self test:&a];
    
    // 需求2:提供一个方法，这个方法的参数是地址，里面不能修改参数的地址。
    [self test1:&a];
}


@end
```

引自：https://www.jianshu.com/p/2fd58ed2cf55

# 2. 内存



