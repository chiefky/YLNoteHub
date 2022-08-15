## ARC 的实现

`ARC`仅仅依靠`LLVM`编译器是无法完成内存管理工作的，它还需要`Runtime`的支持。就比如`__weak`修饰符，如果没有`Runtime`，那么在对象`dealloc`时就不会将`__weak`变量置为`nil`。 `ARC`由以下工具、库来实现：

- clang（LLVM 编译器）3.0 以上
- objc4 Objective-C 运行时库 493.9 以上

## 转换项目时的常见问题

除了以上说明的几点`ARC`新规则以外，`ARC`下还要注意以下几个问题，也是`MRC`转换到`ARC`项目的常见问题：

- `ARC`要求你在`init`方法中将`[super init]`的结果分配给`self`。

```objc
    self = [super init];
    if (self) {
    ...
```

- 你无法实现自定义`retain`或`release`方法。
  实现自定义`retain`或`release`方法会破坏`weak`弱指针。你想要这么做的原因可能如下：
  - ① 性能
    请不要再这样做了，`NSObject`的`retain`和`release`方法的实现现在已经足够快了。如果你仍然发现有问题，请提交错误给苹果。
  - ② 实现自定义`weak`弱指针系统
    请改用`__weak`。
  - ③ 实现单例类
    请改用`shared instance`模式。或者，使用类方法替代实例方法，这样可以避免创建对象。
- “直接赋值” 的实例变量变成强引用了。

在`ARC`之前，实例变量是弱引用（非持有引用） —— 直接将对象分配给实例变量并不延长对象的生命周期。为了使属性变`strong`，你通常会实现或使用`@synthesize`合成 “调用适当内存管理方法” 的访问器方法。相反，有时你为了维持一个弱引用，你可能会像以下实例这样实现访问器方法。

```objc
@interface MyClass : Superclass {
    id thing; // Weak reference.
}
// ...
@end

@implementation MyClass
- (id)thing {
    return thing;
}
- (void)setThing:(id)newThing {
    thing = newThing;
}
// ...
@end
复制代码
```

对于`ARC`，实例变量默认是`strong`强引用 —— 直接将对象分配给实例变量会延长对象的生命周期。迁移工具在将`MRC`代码转换为`ARC`代码时，无法确定它该使用`strong`还是`weak`，所以默认使用`strong`。 若要保持与`MRC`下一致，必须将实例变量使用`__weak`修饰，或使用`weak`关键字的属性。

```objc
@interface MyClass : Superclass {
    id __weak thing;
}
// ...
@end

@implementation MyClass
- (id)thing {
    return thing;
}
- (void)setThing:(id)newThing {
    thing = newThing;
}
// ...
@end
复制代码
```

或者：

```objc
@interface MyClass : Superclass
@property (weak) id thing;
// ...
@end

@implementation MyClass
@synthesize thing;
// ...
@end
复制代码
```

## ARC 补充

### __weak 黑科技

在所有权修饰符中我们简单介绍了`__weak`修饰符。实际上，除了在`MRC`下无法使用`__weak`修饰符以外，还有其他无法使用`__weak`修饰符的情况。

例如，有一些类是不支持`__weak`修饰符的，比如`NSMachPort`。这些类重写了`retain / release`并实现该类独自的引用计数机制。但是赋值以及使用附有`__weak`修饰符的变量都必须恰当地使用 objc4 运行时库中的函数，因此独自实现引用计数机制的类大多不支持`__weak`修饰符。

```objc
    NSMachPort __weak *port = [NSMachPort new]; 
    // 编译错误：Class is incompatible with __weak references 类与弱引用不兼容
复制代码
```

不支持`__weak`修饰符的类，其类的声明中添加了`NS_AUTOMATED_REFCOUNT_WEAK_UNAVAILABLE`宏，该宏的定义如下。

```objc
// Marks classes which cannot participate in the ARC weak reference feature.
#if __has_attribute(objc_arc_weak_reference_unavailable)
#define NS_AUTOMATED_REFCOUNT_WEAK_UNAVAILABLE __attribute__((objc_arc_weak_reference_unavailable))
#else
#define NS_AUTOMATED_REFCOUNT_WEAK_UNAVAILABLE
#endif
复制代码
```

如果将不支持`__weak`的类的对象赋值给`__weak`修饰符的变量，一旦编译器检测出来就会报告编译错误。但是在 Cocoa 框架中，不支持`__weak`修饰符的类极为罕见，因此没有必要太过担心。

`__weak`黑科技来了！！！！！

还有一种情况也不能使用`__weak`修饰符。就是当对象的`allowsWeakReference`/`retainWeakReference`实例方法返回`NO`时，这两个方法的声明如下：

```objc
- (BOOL)allowsWeakReference;
- (BOOL)retainWeakReference;
复制代码
```

这两个方法的默认实现是返回`YES`。

如果我们在类中重写了`allowsWeakReference`方法并返回`NO`，那么如果我们将该类的实例对象赋值给`__weak`修饰符的变量，那么程序就会`Crash`。

例如我们在`HTPerson`类中做了此操作，则以下代码运行就会`Crash`。

```objc
    HTPerson __weak *p = [[HTPerson alloc] init];
复制代码
// 无法对 HTPerson 类的实例持有弱引用。可能是此对象被过度释放，或者正在销毁。
objc[18094]: Cannot form weak reference to instance (0x600001d7c2a0) of class HTPerson. It is possible that this object was over-released, or is in the process of deallocation.
(lldb) 
复制代码
```

所以，对于所有`allowsWeakReference`方法返回`NO`的类的实例都绝对不能使用`__weak`修饰符。

另外，如果实例对象的`retainWeakReference`方法返回`NO`，那么赋值该对象`__weak`修饰符的变量将为`nil`，代表无法通过`__weak`变量访问该对象。

比如以下示例代码：

```objc
    HTPerson *p1 = [[HTPerson alloc] init];    
    HTPerson __weak *p2 = p1;
    NSLog(@"%@", p2);
    NSLog(@"%@", p2);
    NSLog(@"%@", p2);
    NSLog(@"%@", p2);
    NSLog(@"%@", p2);

/* 打印如下：
   <HTPerson: 0x600002e0dd20>
   <HTPerson: 0x600002e0dd20>
   <HTPerson: 0x600002e0dd20>
   <HTPerson: 0x600002e0dd20>
   <HTPerson: 0x600002e0dd20> */
复制代码
```

由于`p1`为`__strong`持有对象的强引用，所以在`p1`作用域结束前，该对象都存在，使用`__weak`修饰的`p2`访问该对象没问题。

下面在`HTPerson`类中重写`retainWeakReference`方法：

```objc
@interface HTPerson ()
{
    NSUInteger count;
}

@implementation HTPerson
- (BOOL)retainWeakReference
{
    if (++count > 3) {
        return NO;
    }
    return [super retainWeakReference];    
}
@end
复制代码
```

再次运行以上代码，发现从第 4 次开始，通过`__weak`变量就无法访问到对象，因为这时候`retainWeakReference`方法返回值为`NO`。

```objc
    HTPerson *p1 = [[HTPerson alloc] init];    
    HTPerson __weak *p2 = p1;
    NSLog(@"%@", p2);
    NSLog(@"%@", p2);
    NSLog(@"%@", p2);
    NSLog(@"%@", p2);
    NSLog(@"%@", p2);

/* 打印如下：
   <HTPerson: 0x600003e23ba0>
   <HTPerson: 0x600003e23ba0>
   <HTPerson: 0x600003e23ba0>
   (null)
   (null) */
复制代码
```

### 查看引用计数

在`ARC`下，我们可以使用`_objc_rootRetainCount`函数查看对象的引用计数。

```objc
uintptr_t _objc_rootRetainCount(id obj);
复制代码
```

但实际上并不能完全信任该函数取得的数值。对于已释放的对象以及不正确的对象地址，有时也返回 “1”。另外，在多线程中使用对象的引用计数数值，因为存在竞争条件的问题，所以取得的数值不一定完全可信。 虽然在调试中`_objc_rootRetainCount`函数很有用，但最好在了解其所具有的问题的基础上来使用。

## 苹果对 ARC 一些问题的回答

> **Q：** 我应该如何看待 ARC ？它将 retains/releases 调用的代码放在哪了？

尝试不要去思考`ARC`将`retains/releases`调用的代码放在哪里，而是思考应用程序算法，思考对象的`strong`和`weak`指针、所有权、以及可能产生的循环引用。

> **Q：** 我还需要为我的对象编写 dealloc 方法吗？

有时候需要。 因为`ARC`不会自动处理`malloc/free`、`Core Foundation`对象的生命周期管理、文件描述符等等，所以你仍然可以通过编写`dealloc`方法来释放这些资源。 你不必（实际上不能）释放实例变量，但可能需要对系统类和其他未使用`ARC`编写的代码调用`[self setDelegate:nil]`。 `ARC`下的`dealloc`方法中不需要且不允许调用`[super dealloc]`，`Runtime`会自动处理。

> **Q：** ARC 中仍然可能存在循环引用吗？

是的，`ARC`自动`retain/release`，也继承了循环引用问题。幸运的是，迁移到`ARC`的代码很少开始泄漏，因为属性已经声明是否`retain`。

> **Q：** block 是如何在 ARC 中工作的？

在`ARC`下，编译器会根据情况自动将栈上的`block`复制到堆上，比如`block`作为函数返回值时，这样你就不必再调用`Block Copy`。

需要注意的一件事是，在`ARC`下，`NSString * __block myString`这样写的话，`block`会对`NSString`对象强引用，而不是造成悬垂指针问题。如果你要和`MRC`保持一致，请使用`__block NSString * __unsafe_unretained myString`或（更好的是）使用`__block NSString * __weak myString`。

> **Q：** 我可以在 ARC 下创建一个 retained 指针的 C 数组吗？

可以，如下示例所示：

```objc
// Note calloc() to get zero-filled memory.
__strong SomeClass **dynamicArray = (__strong SomeClass **)calloc(entries, sizeof(SomeClass *));
for (int i = 0; i < entries; i++) {
     dynamicArray[i] = [[SomeClass alloc] init];
}
 
// When you're done, set each entry to nil to tell ARC to release the object.
for (int i = 0; i < entries; i++) {
     dynamicArray[i] = nil;
}
free(dynamicArray);
复制代码
```

这里有一些注意点：

- 在某些情况下，你需要编写`__strong SomeClass **`，因为默认是`__autoreleasing SomeClass **`。
- 分配的内存区域必须初始化为 0（`zero-filled`）。
- 在`free`数组之前，必须将每个元素赋值为`nil`（`memset`或`bzero`将不起作用）。
- 你应该避免使用`memcpy`或`realloc`。

> **Q：** ARC 速度上慢吗？

不。编译器有效地消除了许多无关的`retain/release`调用，并且已经投入了大量精力来加速 Objective-C 运行时。特别的是，当方法的调用者是`ARC`代码时，常见的 “`return a retain/autoreleased object`” 模式要快很多，并且实际上并不将对象放入自动释放池中。

需要注意的一个问题是，优化器不是在常见的调试配置中运行的，所以预计在`-O0`模式下将会比`-Os`模式下看到更多的`retain/release`调用。

> **Q：** ARC 在 ObjC++ 模式下工作吗？

是。你甚至可以在类和容器中放置`strong/weak`的`id`对象。`ARC`编译器会在复制构造函数和析构函数等中合成`retain/release`逻辑以使其运行。

> **Q：** 哪些类不支持 weak 弱引用？

你当前无法创建对以下类的实例的`weak`弱引用：NSATSTypesetter、NSColorSpace、NSFont、NSMenuView、NSParagraphStyle、NSSimpleHorizontalTypesetter 和 NSTextView。

**注意：** 此外，在 OS X v10.7 中，你无法创建对 NSFontManager，NSFontPanel、NSImage、NSTableCellView、NSViewController、NSWindow 和 NSWindowController 实例的`weak`弱引用。此外，在 OS X v10.7 中，AV Foundation 框架中的任何类都不支持`weak`弱引用。

此外，你无法在`ARC`下创建 NSHashTable、NSMapTable 和 NSPointerArray 类的实例的`weak`弱引用。

> **Q：** 当我继承一个使用了 NSCopyObject 的类，如 NSCell 时，我需要做些什么？

没什么特别的。`ARC`会关注以前必须显式添加额外`retain`的情况。使用`ARC`，所有的复制方法只需要复制实例变量就可以了。

> **Q：** 我可以对指定文件选择退出`ARC`而使用`MRC`吗？

可以。当你迁移项目到`ARC`或创建一个`ARC`项目时，所以`Objective-C`源文件的默认编译器标志将设置为`-fobjc-arc`，你可以使用`-fno-objc-arc`编译器标志为指定的类禁用`ARC`。操作如下图所示：

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1dd790dabea04b83a407c6feafb7c246~tplv-k3u1fbpfcp-zoom-1.image)

> **Q：** 在 Mac 上是否弃用了 GC (Garbage Collection) 机制？

OS X Mountain Lion v10.8 中不推荐使用`GC`机制，并且将在 OS X 的未来版本中删除`GC`机制。`ARC`是推荐的替代技术。为了帮助现有应用程序迁移，Xcode 4.3 及更高版本中的`ARC`迁移工具支持将使用`GC`的 OS X 应用程序迁移到`ARC`。

**注意：** 对于面向 Mac App Store 的应用，Apple 强烈建议你尽快使用`ARC`替换`GC`，因为 Mac App Store Guidelines 禁止使用已弃用的技术，否则不会通过审核，详情请参阅 [Mac App Store Review Guidelines](http://developer.apple.com/appstore/mac/resources/approval/guidelines.html)。