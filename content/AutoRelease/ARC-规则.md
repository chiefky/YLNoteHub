#  iOS - 老生常谈内存管理（三）：ARC 面世

## 前言

  `ARC`全称`Automatic Reference Counting`，自动引用计数内存管理，是苹果在 iOS 5、OS X Lion 引入的新的内存管理技术。`ARC`是一种编译器功能，它通过`LLVM`编译器和`Runtime`协作来进行自动管理内存。`LLVM`编译器会在编译时在合适的地方为 OC 对象插入`retain`、`release`和`autorelease`代码来自动管理对象的内存，省去了在`MRC`手动引用计数下手动插入这些代码的工作，减轻了开发者的工作量，让开发者可以专注于应用程序的代码、对象图以及对象间的关系上。
  本文通过讲解`MRC`到`ARC`的转变、`ARC`规则以及使用注意，来帮助大家掌握`iOS`的内存管理。
  下图是苹果官方文档给出的从`MRC`到`ARC`的转变。

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cf76b8af9fbe4dbcb4ae9da633b3b4e3~tplv-k3u1fbpfcp-zoom-1.image)

## 摘要

`ARC`的工作原理是在编译时添加相关代码，以确保对象能够在必要时存活，但不会一直存活。从概念上讲，它通过为你添加适当的内存管理方法调用来遵循与`MRC`相同的内存管理规则。

为了让编译器生成正确的代码，`ARC`限制了一些方法的使用以及你使用桥接(`toll-free bridging`)的方式，请参阅[ Managing Toll-Free Bridging ](https://juejin.cn/post/6844904130431942670#heading-31)章节。`ARC`还为对象引用和属性声明引入了新的生命周期修饰符。

`ARC`在`Xcode 4.2 for OS X v10.6 and v10.7 (64-bit applications) `以及`iOS 4 and iOS 5`应用程序中提供支持。但`OS X v10.6 and iOS 4`不支持`weak`弱引用。

Xcode 提供了一个迁移工具，可以自动将`MRC`代码转换为`ARC`代码（如删除`retain`和`release`调用），而不用重新再创建一个项目（选择 Edit > Convert > To Objective-C ARC）。迁移工具会将项目中的所有文件转换为使用`ARC`的模式。如果对于某些文件使用`MRC`更方便的话，你可以选择仅在部分文件中使用`ARC`。

## ARC 概述

`ARC`会分析对象的生存期需求，并在编译时自动插入适当的内存管理方法调用的代码，而不需要你记住何时使用`retain`、`release`、`autorelease`方法。编译器还会为你生成合适的`dealloc`方法。一般来说，如果你使用`ARC`，那么只有在需要与使用`MRC`的代码进行交互操作时，传统的 Cocoa 命名约定才显得重要。

Person 类的完整且正确的实现可能如下所示：

```objc
@interface Person : NSObject
@property NSString *firstName;
@property NSString *lastName;
@property NSNumber *yearOfBirth;
@property Person *spouse;
@end
 
@implementation Person
@end
复制代码
```

默认情况下，对象属性是`strong`。关于`strong`请参阅[ 所有权修饰符 ](https://juejin.cn/post/6844904130431942670#heading-12)章节。

使用`ARC`，你可以这样实现 contrived 方法，如下所示：

```objc
- (void)contrived {
    Person *aPerson = [[Person alloc] init];
    [aPerson setFirstName:@"William"];
    [aPerson setLastName:@"Dudney"];
    [aPerson setYearOfBirth:[[NSNumber alloc] initWithInteger:2011]];
    NSLog(@"aPerson: %@", aPerson);
}
复制代码
```

`ARC`会负责内存管理，因此 Person 和 NSNumber 对象都不会泄露。

你还可以这样安全地实现 Person 的 takeLastNameFrom: 方法，如下所示：

```objc
- (void)takeLastNameFrom:(Person *)person {
    NSString *oldLastname = [self lastName];
    [self setLastName:[person lastName]];
    NSLog(@"Lastname changed from %@ to %@", oldLastname, [self lastName]);
}
复制代码
```

`ARC`会确保在 NSLog 语句之前不释放 oldLastName 对象。

### ARC 实施新规则

`ARC`引入了一些在使用其他编译器模式时不存在的新规则。这些规则旨在提供完全可靠的内存管理模型。有时候，它们直接地带来了最好的实践体验，也有时候它们简化了代码，甚至在你丝毫没有关注内存管理问题的时候帮你解决了问题。在`ARC`下必须遵守以下规则，如果违反这些规则，就会编译错误。

- 不能使用 retain / release / retainCount / autorelease
- 不能使用 NSAllocateObject / NSDeallocateObject
- 须遵守内存管理的方法命名规则
- 不能显式调用 dealloc
- 使用 @autoreleasepool 块替代 NSAutoreleasePool
- 不能使用区域（NSZone）
- 对象型变量不能作为 C 语言结构体（struct / union）的成员
- 显式转换 “id” 和 “void *” —— 桥接

#### 不能使用 retain / release / retainCount / autorelease

在`ARC`下，禁止开发者手动调用这些方法，也禁止使用`@selector(retain)`，`@selector(release) `等，否则编译不通过。但你仍然可以对 Core Foundation 对象使用`CFRetain`、`CFRelease`等相关函数（请参阅`《Managing Toll-Free Bridging》`章节）。

#### 不能使用 NSAllocateObject / NSDeallocateObject

在`ARC`下，禁止开发者手动调用这些函数，否则编译不通过。 你可以使用`alloc`创建对象，而`Runtime`会负责`dealloc`对象。

#### 须遵守内存管理的方法命名规则

在`MRC`下，通过 `alloc / new / copy / mutableCopy` 方法创建对象会直接持有对象，我们定义一个 “创建并持有对象” 的方法也必须以 `alloc / new / copy / mutableCopy` 开头命名，并且必须返回给调用方所应当持有的对象。如果在`ARC`下需要与使用`MRC`的代码进行交互，则也应该遵守这些规则。

为了允许与`MRC`代码进行交互操作，`ARC`对方法命名施加了约束： 访问器方法的方法名不能以`new`开头。这意味着你不能声明一个名称以`new`开头的属性，除非你指定一个不同的`getterName`：

```objc
// Won't work:
@property NSString *newTitle;
 
// Works:
@property (getter = theNewTitle) NSString *newTitle;
复制代码
```

#### 不能显式调用 dealloc

无论在`MRC`还是`ARC`下，当对象引用计数为 0，系统就会自动调用`dealloc`方法。大多数情况下，我们会在`dealloc`方法中移除通知或观察者对象等。

在`MRC`下，我们可以手动调用`dealloc`。但在`ARC`下，这是禁止的，否则编译不通过。

在`MRC`下，我们实现`dealloc`，必须在实现末尾调用`[super dealloc]`。

```objc
// MRC
- (void)dealloc
{
    // 其他处理
    [super dealloc];
}
复制代码
```

而在`ARC`下，`ARC`会自动对此处理，因此我们不必也禁止写`[super dealloc]`，否则编译错误。

```objc
// ARC
- (void)dealloc
{
    // 其他处理
    [super dealloc]; // 编译错误：ARC forbids explicit message send of 'dealloc'
}
复制代码
```

#### 使用 @autoreleasepool 块替代 NSAutoreleasePool

在`ARC`下，自动释放池应使用`@autoreleasepool`，禁止使用`NSAutoreleasePool`，否则编译错误。

```objc
NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init]; 
// error：'NSAutoreleasePool' is unavailable: not available in automatic reference counting mode
复制代码
```

> 关于`@autoreleasepool`的原理，可以参阅[《iOS - 聊聊 autorelease 和 @autoreleasepool》](https://juejin.im/post/6844904094503567368)。

#### 不能使用区域（NSZone）

对于现在的运行时系统（编译器宏 __ OBJC2 __ 被设定的环境），不管是`MRC`还是`ARC`下，区域（NSZone）都已单纯地被忽略。

> **NSZone：** 摘自《Objective-C 高级编程：iOS 与 OS X 多线程和内存管理》
>
> NSZone 是为防止内存碎片化而引入的结构。对内存分配的区域本身进行多重化管理，根据使用对象的目的、对象的大小分配内存，从而提高了内存管理的效率。
> 但是，现在的运行时系统已经忽略了区域的概念。运行时系统中的内存管理本身已极具效率，使用区域来管理内存反而会引起内存使用效率低下以及源代码复杂化等问题。
> 下图是使用多重区域防止内存碎片化的例子： ![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d69ee2a30d5b4fe29bd9d1313d13a1cb~tplv-k3u1fbpfcp-zoom-1.image)

#### 对象型变量不能作为 C 语言结构体（struct / union）的成员

C 语言的结构体（struct / union）成员中，如果存在 Objective-C 对象型变量，便会引起编译错误。

> **备注：** Xcode10 开始支持在 ARC 模式下在 C Struct 里面引用 Objective-C 对象。之前可以用 Objective-C++。

```objc
struct Data {
    NSMutableArray *mArray;
};
// error：ARC forbids Objective-C objs in struct or unions NSMutableArray *mArray;
复制代码
```

虽然是 LLVM 编译器 3.0，但不论怎样，C 语言的规约上没有方法来管理结构体成员的生存周期。因为`ARC`把内存管理的工作分配给编译器，所以编译器必须能够知道并管理对象的生存周期。例如 C 语言的自动变量（局部变量）可使用该变量的作用域管理对象。但是对于 C 语言的结构体成员来说，这在标准上就是不可实现的。因此，必须要在结构体释放之前将结构体中的对象类型的成员释放掉，但是编译器并不能可靠地做到这一点，所以对象型变量不能作为 C 语言结构体的成员。

这个问题有以下三种解决方案：

- ① 使用 Objective-C 对象替代结构体。这是最好的解决方案。

如果你还是坚持使用结构体，并把对象型变量加入到结构体成员中，可以使用以下两种方案：

- ② 将 Objective-C 对象通过`Toll-Free Bridging`强制转换为`void *`类型，请参阅`《Managing Toll-Free Bridging》`章节。
- ③ 对 Objective-C 对象附加`__unsafe_unretained`修饰符。

```objc
struct Data {
    NSMutableArray __unsafe_unretained *mArray;
};
复制代码
```

附有`__unsafe_unretained`修饰符的变量不属于编译器的内存管理对象。如果管理时不注意赋值对象的所有者，便有可能遭遇内存泄漏或者程序崩溃。这点在使用时应多加注意。

```objc
struct x { NSString * __unsafe_unretained S; int X; }
复制代码
```

`__unsafe_unretained`指针在对象被销毁后是不安全的，但它对诸如字符串常量之类的从一开始就确定永久存活的对象非常有用。

#### 显式转换 “id” 和 “void *” —— 桥接

在`MRC`下，我们可以直接在 `id` 和 `void *` 变量之间进行强制转换。

```objc
    id obj = [[NSObject alloc] init];
    void *p = obj;
    id o = p;
    [o release];
复制代码
```

但在`ARC`下，这样会引起编译报错：在`Objective-C`指针类型`id`和`C`指针类型`void *`之间进行转换需要使用`Toll-Free Bridging`，请参阅[ Managing Toll-Free Bridging ](https://juejin.cn/post/6844904130431942670#heading-31)章节。

```objc
    id obj = [[NSObject alloc] init];
    void *p = obj; // error：Implicit conversion of Objective-C pointer type 'id' to C pointer type 'void *' requires a bridged cast
    id o = p;      // error：Implicit conversion of C pointer type 'void *' to Objective-C pointer type 'id' requires a bridged cast
    [o release];   // error：'release' is unavailable: not available in automatic reference counting mode
复制代码
```

