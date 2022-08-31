### 所有权修饰符

`ARC`为对象引入了几个新的生命周期修饰符（我们称为 “所有权修饰符”）以及弱引用功能。弱引用`weak`不会延长它指向的对象的生命周期，并且该对象没有强引用（即`dealloc`）时自动置为`nil`。 
你应该利用这些修饰符来管理程序中的对象图。特别是，`ARC`不能防止强引用循环（以前称为`Retain Cycles`，请参阅[《从 MRC 说起 —— 使用弱引用来避免 Retain Cycles》](https://juejin.cn/post/6844904129676984334#heading-17)章节）。明智地使用弱引用`weak`将有助于确保你不会创建循环引用。

#### 属性关键字

`ARC`中引入了新的属性关键字`strong`和`weak`，如下所示：

```objc
// 以下声明同：@property(retain) MyClass *myObject;
@property(strong) MyClass *myObject;
 
// 以下声明类似于：@property（assign）MyClass *myObject；
// 不同的是，如果 MyClass 实例被释放，属性值赋值为 nil，而不像 assign 一样产生悬垂指针。
@property(weak) MyClass *myObject;
复制代码
```

`strong`和`weak`属性关键字分别对应`__strong`和`__weak`所有权修饰符。在`ARC`下，`strong`是对象类型的属性的默认关键字。

在`ARC`中，对象类型的变量都附有所有权修饰符，总共有以下 4 种。

```objc
__strong
__weak
__unsafe_unretained
__autoreleasing
复制代码
```

- `__strong`是默认修饰符。只要有强指针指向对象，对象就会保持存活。
- `__weak`指定一个不使引用对象保持存活的引用。当一个对象没有强引用时，弱引用`weak`会自动置为`nil`。
- `__unsafe_unretained`指定一个不使引用对象保持存活的引用，当一个对象没有强引用时，它不会置为`nil`。如果它引用的对象被销毁，就会产生悬垂指针。
- `__autoreleasing`用于表示通过引用（`id *`）传入，并在返回时（`autorelease`）自动释放的参数。

在对象变量的声明中使用所有权修饰符时，正确的格式为：

```objc
    ClassName * qualifier variableName;
复制代码
```

例如：

```objc
    MyClass * __weak myWeakReference;
    MyClass * __unsafe_unretained myUnsafeReference;
复制代码
```

其它格式在技术上是不正确的，但编译器会 “原谅”。也就是说，以上才是标准写法。

#### __strong

`__strong`修饰符为强引用，会持有对象，使其引用计数 +1。该修饰符是对象类型变量的默认修饰符。如果我们没有明确指定对象类型变量的所有权修饰符，其默认就为`__strong`修饰符。

```objc
    id obj = [NSObject alloc] init];
    // -> id __strong obj = [NSObject alloc] init];
复制代码
```

#### __weak

如果单单靠`__strong`完成内存管理，那必然会发生循环引用的情况造成内存泄漏，这时候`__weak`就出来解决问题了。 `__weak`修饰符为弱引用，不会持有对象，对象的引用计数不会增加。`__weak`可以用来防止循环引用。

以下单纯地使用`__weak`修饰符修饰变量，编译器会给出警告，因为`NSObject`的实例创建出来没有强引用，就会立即释放。

```objc
    id __weak weakObj = [[NSObject alloc] init]; // ⚠️Assigning retained object to weak variable; object will be released after assignment
    NSLog(@"%@", obj);
    //  (null)
复制代码
```

以下`NSObject`的实例已有强引用，再赋值给`__weak`修饰的变量就不会有警告了。

```objc
    id __strong strongObj = [[NSObject alloc] init];
    id __weak weakObj = strongObj;
复制代码
```

当对象被`dealloc`时，指向该对象的`__weak`变量会被赋值为`nil`。（具体的执行过程可以参阅：[《iOS - 老生常谈内存管理（四）：源码分析内存管理方法》](https://juejin.cn/post/6844904131719593998)）

> **备注：**`__weak`仅在`ARC`中才能使用，在`MRC`中是使用`__unsafe_unretained`修饰符来代替。

#### __unsafe_unretained

`__unsafe_unretained`修饰符的特点正如其名所示，不安全且不会持有对象。

> **注意：** 尽管`ARC`内存管理是编译器的工作，但是附有`__unsafe_unretained`修饰符的变量不属于编译器的内存管理对象。这一点在使用时要注意。

“不会持有对象” 这一特点使它和`__weak`的作用相似，可以防止循环引用。 
“不安全“ 这一特点是它和`__weak`的区别，那么它不安全在哪呢？

我们来看代码：

```objc
    id __weak weakObj = nil;
    id __unsafe_unretained uuObj = nil;
    {
        id __strong strongObj = [[NSObject alloc] init];
        weakObj = strongObj;
        unsafeUnretainedObj = strongObj;
        NSLog(@"strongObj:%@", strongObj);
        NSLog(@"weakObj:%@", weakObj);
        NSLog(@"unsafeUnretainedObj:%@", unsafeUnretainedObj);
    }
    NSLog(@"-----obj dealloc-----");
    NSLog(@"weakObj:%@", weakObj);
    NSLog(@"unsafeUnretainedObj:%@", unsafeUnretainedObj); // Crash:EXC_BAD_ACCESS

/*
strongObj:<NSObject: 0x6000038f4340>
weakObj:<NSObject: 0x6000038f4340>
unsafeUnretainedObj:<NSObject: 0x6000038f4340>
-----obj dealloc-----
weakObj:(null)
(lldb) 
*/
复制代码
```

以上代码运行崩溃。原因是`__unsafe_unretained`修饰的对象在被销毁之后，指针仍然指向原对象地址，我们称它为 “悬垂指针”。这时候如果继续通过指针访问原对象的话，就会导致`Crash`。而`__weak`修饰的对象在被释放之后，会将指向该对象的所有`__weak`指针变量全都置为`nil`。这就是`__unsafe_unretained`不安全的原因。所以，在使用`__unsafe_unretained`修饰符修饰的对象时，需要确保它未被销毁。

> **Q：** 既然 __weak 更安全，那么为什么已经有了 __weak 还要保留 __unsafe_unretained ？
>
> - `__weak`仅在`ARC`中才能使用，而`MRC`只能使用`__unsafe_unretained`；
> - `__unsafe_unretained`主要跟 C 代码交互；
> - `__weak`对性能会有一定的消耗，当一个对象`dealloc`时，需要遍历对象的`weak`表，把表里的所有`weak`指针变量值置为`nil`，指向对象的`weak`指针越多，性能消耗就越多。所以`__unsafe_unretained`比`__weak`快。当明确知道对象的生命周期时，选择`__unsafe_unretained`会有一些性能提升。
>
> 
> A 持有 B 对象，当 A 销毁时 B 也销毁。这样当 B 存在，A 就一定会存在。而 B 又要调用 A 的接口时，B 就可以存储 A 的`__unsafe_unretained`指针。 比如，MyViewController 持有 MyView，MyView 需要调用 MyViewController 的接口。MyView 中就可以存储`__unsafe_unretained MyViewController *_viewController`。 
>
> 虽然这种性能上的提升是很微小的。但当你很清楚这种情况下，`__unsafe_unretained`也是安全的，自然可以快一点就是一点。而当情况不确定的时候，应该优先选用`__weak`。

#### __autoreleasing

##### 自动释放池

首先讲一下自动释放池，在`ARC`下已经禁止使用`NSAutoreleasePool`类创建自动释放池，而用`@autoreleasepool`替代。

- `MRC`下可以使用`NSAutoreleasePool`或者`@autoreleasepool`。建议使用`@autoreleasepool`，苹果说它比`NSAutoreleasePool`快大约六倍。

```objc
    NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
    // Code benefitting from a local autorelease pool.
    [pool release]; // [pool drain]
复制代码
```

> **Q：** 释放`NSAutoreleasePool`对象，使用`[pool release]`与`[pool drain]`的区别？
>
>  Objective-C 语言本身是支持 GC 机制的，但有平台局限性，仅限于 MacOS 开发中，iOS 开发用的是 RC 机制。在 iOS 的 RC 环境下`[pool release]`和`[pool drain]`效果一样，但在 GC 环境下`drain`会触发 GC 而`release`不做任何操作。使用`[pool drain]`更佳，一是它的功能对系统兼容性更强，二是这样可以跟普通对象的`release`区别开。（注意：苹果已在 OS X Mountain Lion v10.8 中弃用`GC`机制，而使用`ARC`替代）

- `ARC`下只能使用`@autoreleasepool`。

```objc
    @autoreleasepool {
        // Code benefitting from a local autorelease pool.
    }
复制代码
```

> 关于`@autoreleasepool`的底层原理，可以参阅[《iOS - 聊聊 autorelease 和 @autoreleasepool》](https://juejin.im/post/6844904094503567368)。

##### __autoreleasing 使用

在`MRC`中我们可以给对象发送`autorelease`消息来将它注册到`autoreleasepool`中。而在`ARC`中`autorelease`已禁止调用，我们可以使用`__autoreleasing`修饰符修饰对象将对象注册到`autoreleasepool`中。

```objc
    @autoreleasepool {
        id __autoreleasing obj = [[NSObject alloc] init];
    }
复制代码
```

以上代码在`MRC`中等价于:

```objc
    NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
    id obj = [[NSObject alloc] init];
    [obj autorelease];
    [pool drain];
    // 或者
    @autoreleasepool {
        id obj = [[NSObject alloc] init];
        [obj autorelease];
    }
复制代码
```

##### __autoreleasing 是二级指针类型的默认修饰符

前面我们说过，对象指针的默认所有权修饰符是`__strong`。 而二级指针类型（`ClassName **`或`id *`）的默认所有权修饰符是`__autoreleasing`。如果我们没有明确指定二级指针类型的所有权修饰符，其默认就会附加上`__autoreleasing`修饰符。

比如，我们经常会在开发中使用到`NSError`打印错误信息，我们通常会在方法的参数中传递`NSError`对象的指针。如`NSString`的`stringWithContentsOfFile`类方法，其参数`NSError **`使用的是`__autoreleasing`修饰符。

```objc
    NSString *str = [NSString stringWithContentsOfFile:<#(nonnull NSString *)#>
                                              encoding:<#(NSStringEncoding)#> 
                                              error:<#(NSError *__autoreleasing  _Nullable * _Nullable)#>];
复制代码
```

示例：我们声明一个参数为`NSError **`的方法，但不指定其所有权修饰符。

```objc
- (BOOL)performOperationWithError:(NSError **)error;
复制代码
```

接着我们尝试调用该方法，发现智能提示中的参数`NSError **`附有`__autoreleasing`修饰符。可见，如果我们没有明确指定二级指针类型的所有权修饰符，其默认就会附加上`__autoreleasing`修饰符。

```objc
    NSError *error = nil;
    BOOL result = [self performOperationWithError:<#(NSError *__autoreleasing *)#>];
复制代码
```

##### 注意

需要注意的是，赋值给二级指针类型时，所有权修饰符必须一致，否则会编译错误。

```objc
    NSError *error = nil;
    NSError **error1 = &error;                 // error：Pointer to non-const type 'NSError *' with no explicit ownersh
    NSError *__autoreleasing *error2 = &error; // error：Initializing 'NSError *__autoreleasing *' with an expression of type 'NSError *__strong *' changes retain/release properties of pointer
    NSError *__weak *error3 = &error;          // error：Initializing 'NSError *__weak *' with an expression of type 'NSError *__strong *' changes retain/release properties of pointer
    NSError *__strong *error4 = &error;        // 编译通过
复制代码
    NSError *__weak error = nil;
    NSError *__weak *error1 = &error;          // 编译通过
复制代码
    NSError *__autoreleasing error = nil;
    NSError *__autoreleasing *error1 = &error; // 编译通过
复制代码
```

我们前面说过，二级指针类型的默认修饰符是`__autoreleasing`。那为什么我们调用方法传入`__strong`修饰的参数就可以编译通过呢？

```objc
    NSError *__strong error = nil;
    BOOL result = [self performOperationWithError:<#(NSError *__autoreleasing *)#>];
复制代码
```

其实，编译器自动将我们的代码转化成了以下形式：

```objc
    NSError *__strong error = nil;
    NSError *__autoreleasing tmp = error;
    BOOL result = [self performOperationWithError:&tmp];
    error = tmp;
复制代码
```

可见，当局部变量声明（`__strong`）和参数（`__autoreleasing`）之间不匹配时，会导致编译器创建临时变量。你可以将显示地指定局部变量所有权修饰符为`__autoreleasing`~~或者不显式指定（因为其默认就为`__autoreleasing`）~~，或者显示地指定参数所有权修饰符为`__strong`，来避免编译器创建临时变量。

```objc
- (BOOL)performOperationWithError:(NSError *__strong *)error;
复制代码
```

但是在`MRC`引用计数内存管理规则中：使用`alloc/new/copy/mutableCopy`等方法创建的对象，创建并持有对象；其他情况创建对象但并不持有对象。由谁创建就由谁负责释放，很明显这里的 NSError为了在使用参数获得对象时，遵循此规则，我们应该指定二级指针类型参数修饰符为`__autoreleasing`。

在`《从 MRC 说起 —— 你不持有通过引用返回的对象》`章节中也说到，Cocoa 中的一些方法指定通过引用返回对象（即，它们采用`ClassName **`或`id *`类型的参数），常见的就是使用`NSError`对象。当你调用这些方法时，你不会创建该`NSError`对象，因此你不持有该对象，也无需释放它。而`__strong`代表持有对象，因此应该使用`__autoreleasing`。

另外，我们在显示指定`__autoreleasing`修饰符时，必须注意对象变量要为自动变量（包括局部变量、函数以及方法参数），否则编译不通过。

```objc
    static NSError __autoreleasing *error = nil; // Global variables cannot have __autoreleasing ownership
复制代码
```

#### 使用所有权修饰符来避免循环引用

前面已经说过`__weak`和`__unsafe_unretained `修饰符可以用来循环引用，这里再来啰嗦几句。

如果两个对象互相强引用，就产生了循环引用，导致两个对象都不能被销毁，内存泄漏。或者多个对象，每个对象都强引用下一个对象直到回到第一个，产生大环循环引用，这些对象也均不能被销毁。

> 在`ARC`中，“循环引用” 是指两个对象都通过`__strong`持有对方。

解决 “循环引用” 问题就是采用 “断环” 的方式，让其中一方持有另一方的弱引用。同`MRC`，父对象对它的子对象持有强引用，而子对象对父对象持有弱引用。

> 在`ARC`中，“弱引用” 是指`__weak`或`__unsafe_unretained `。

##### delegate 避免循环引用

`delegate`避免循环引用，就是在委托方声明`delegate`属性时，使用`weak`关键字。

```objc
@property (nonatomic, weak) id<protocolName> delegate;
复制代码
```

##### block 避免循环引用

> **Q：** 为什么 block 会产生循环引用？

- ① 相互循环引用： 如果当前`block`对当前对象的某一成员变量进行捕获的话，可能会对它产生强引用。根据`block`的变量捕获机制，如果`block`被拷贝到堆上，且捕获的是对象类型的`auto`变量，则会连同其所有权修饰符一起捕获，所以如果对象是`__strong`修饰，则`block`会对它产生强引用（如果`block`在栈上就不会强引用）。而当前`block`可能又由于当前对象对其有一个强引用，就产生了相互循环引用的问题；
- ② 大环引用： 我们如果使用`__block`的话，在`ARC`下可能会产生循环引用（`MRC`则不会）。由于`__block`修饰符会将变量包装成一个对象，如果`block`被拷贝到堆上，则会直接对`__block`变量产生强引用，而`__block`如果修饰的是对象的话，会根据对象的所有权修饰符做出相应的操作，形成强引用或者弱引用。如果对象是`__strong`修饰（如`__block id x`），则`__block`变量对它产生强引用（在`MRC`下则不会），如果这时候该对象是对`block`持有强引用的话，就产生了大环引用的问题。在`ARC`下可以通过断环的方式去解除循环引用，可以在`block`中将指针置为`nil`（`MRC`不会循环引用，则不用解决）。但是有一个弊端，如果该`block`一直得不到调用，循环引用就一直存在。

**ARC 下的解决方式：**

- 用`__weak`或者`__unsafe_unretained`解决：

```objc
    __weak typeof(self) weakSelf = self;
    self.block = ^{
        NSLog(@"%p",weakSelf);
    };
复制代码
    __unsafe_unretained id uuSelf = self;
    self.block = ^{
        NSLog(@"%p",uuSelf);
    };
复制代码
```

![img](https:////p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/961eb7c4378741698b08abc341fa02c5~tplv-k3u1fbpfcp-zoom-1.image)

> **注意**：`__unsafe_unretained`会产生悬垂指针，建议使用`weak`。

 对于 non-trivial cycles，我们需要这样做：

```objc
    __weak typeof(self) weakSelf = self;
    self.block = ^{
        __strong typeof(weakSelf) strongSelf = weakSelf;
        if(!strongSelf) return;
        NSLog(@"%p",weakSelf);
    };
复制代码
```

- 用`__block`解决（必须要调用`block`）：

缺点：必须要调用`block`，而且`block`里要将指针置为`nil`。如果一直不调用`block`，对象就会一直保存在内存中，造成内存泄漏。

```objc
    __block id blockSelf = self;
    self.block = ^{
        NSLog(@"%p",blockSelf);
        blockSelf = nil;
    };
    self.block();
复制代码
```

![img](https:////p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3beaf083a8e4469fb31f4814bd711460~tplv-k3u1fbpfcp-zoom-1.image)

**MRC 下的解决方式：**

- 用`__unsafe_unretained`解决：同`ARC`。
- 用`__block`解决（在`MRC`下使用`__block`修饰对象类型，在`block`内部不会对该对象进行`retain`操作，所以在`MRC`环境下可以通过`__block`解决循环引用的问题）

```objc
    __block id blockSelf = self;
    self.block = ^{
        NSLog(@"%p",blockSelf);
    };
复制代码
```

更多关于`block`的内容，可以参阅[《OC - Block 详解》](https://juejin.im/post/6844904070746996750#heading-2)。

### 属性

说到属性，不得不提一下`@synthesize`和`@dynamic`这两个指令。

#### @synthesize 和 @dynamic

- `@property`：帮我们自动生成属性的`setter`和`getter`方法的声明。
- `@synthesize`：帮我们自动生成`setter`和`getter`方法的实现以及下划线成员变量。

以前我们需要手动对每个`@property`添加`@synthesize`，而在 iOS 6 之后 LLVM 编译器引入了 “`property autosynthesis`”，即属性自动合成。换句话说，就是编译器会自动为每个`@property`添加`@synthesize`。

> **Q：** `@synthesize`现在有什么作用呢？
>
>  如果我们同时重写了`setter`和`getter`方法，则编译器就不会为这个`@property`添加`@synthesize`，这时候就不存在下划线成员变量，所以我们需要手动添加`@synthesize`。
>
> ```objc
> @synthesize propertyName = _propertyName;
> 复制代码
> ```

有时候我们不希望编译器为我们`@synthesize`，我们希望在程序运行过程中再去决定该属性存取方法的实现，就可以使用`@dynamic`。

- `@dynamic` ：告诉编译器不用自动进行`@synthesize`，等到运行时再添加方法实现，但是它不会影响`@property`生成的`setter`和`getter`方法的声明。`@dynamic`是 OC 为动态运行时语言的体现。动态运行时语言与编译时语言的区别：动态运行时语言将函数决议推迟到运行时，编译时语言在编译器进行函数决议。

```objc
@dynamic propertyName;
复制代码
```

#### 属性“内存管理”关键字与所有权修饰符的对应关系

| 属性“内存管理”关键字 | 所有权修饰符        |
| -------------------- | ------------------- |
| assign               | __unsafe_unretained |
| unsafe_unretained    | __unsafe_unretained |
| weak                 | __weak              |
| retain               | __strong            |
| strong               | __strong            |
| copy                 | __strong            |

更多关于属性关键字的内容，可以参阅[《OC - 属性关键字和所有权修饰符》](https://juejin.im/post/6844904067425124366)。

### 管理 Outlets 的模式在 iOS 和 OS X 平台下变得一致

在`ARC`下，`iOS`和`OS X`平台中声明`outlets`的模式变得一致。你应该采用的模式为：在`nib`或者`storyboard`中，除了来自文件所有者的`top-level`对象的`outlets`应该使用`strong`，其它情况下应该使用`weak`修饰`outlets`。（详情见 [Nib Files](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/LoadingResources/CocoaNibs/CocoaNibs.html#//apple_ref/doc/uid/10000051i-CH4) in [Resource Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/LoadingResources/Introduction/Introduction.html#//apple_ref/doc/uid/10000051i)）

### 栈变量初始化为 nil

使用`ARC`，`strong`、`weak`和`autoreleasing`的栈变量现在会默认初始化为`nil`。例如：

```objc
- (void)myMethod {
    NSString *name;
    NSLog(@"name: %@", name);
}
复制代码
```

打印`name`的值为`null`，而不是程序`Crash`。

### 使用编译器标志启用和禁用 ARC

使用`-fobjc-arc`编译器标志启用`ARC`。如果对你来说，某些文件使用`MRC`更方便，那你可以仅对部分文件使用`ARC`。对于使用`ARC`作为默认方式的项目，可以使用`-fno-objc-arc`编译器标志为指定文件禁用`ARC`。如下图所示：

![img](https:////p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/01353d2ec5c348849c8d55764bb6b55b~tplv-k3u1fbpfcp-zoom-1.image)

`ARC`支持 Xcode 4.2 及更高版本、OS X v10.6 及更高版本 (64-bit applications) 、iOS 4 及更高版本。但 OS X v10.6 和 iOS 4 不支持`weak`弱引用。Xcode 4.1 及更早版本中不支持`ARC`。

## 


