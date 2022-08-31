## Managing Toll-Free Bridging

### Toll-Free Bridging

你在项目中可能会使用到`Core Foundation`样式的对象，它可能来自`Core Foundation`框架或者采用`Core Foundation`约定标准的其它框架如`Core Graphics`。

编译器不会自动管理`Core Foundation`对象的生命周期，你必须根据`Core Foundation`内存管理规则调用`CFRetain`和`CFRelease`。请参阅[《Memory Management Programming Guide for Core Foundation》](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFMemoryMgmt/CFMemoryMgmt.html#//apple_ref/doc/uid/10000127i)。

在`MRC`下，我们可以直接在`Objective-C`指针类型`id`和`C`指针类型`void *`之间进行强制转换，如`Foundation`对象和`Core Foundation`对象进行转换。由于都是手动管理内存，无须关心内存管理权的移交问题。

在`ARC`下，进行`Foundation`对象和`Core Foundation`对象的类型转换，需要使用`Toll-Free Bridging`（`桥接`）告诉编译器对象的所有权语义。你要选择使用`__bridge`、`__bridge_retained`、`__bridge_transfer`这三种`桥接`方案中的一种来确定对象的内存管理权移交问题，它们的作用分别如下：

| 桥接方案                                 | 用法     | 内存管理权                                                   | 引用计数         |
| ---------------------------------------- | -------- | ------------------------------------------------------------ | ---------------- |
| __bridge                                 | F <=> CF | 不改变                                                       | 不改变           |
| __bridge_retained (或 CFBridgingRetain)  | F => CF  | ARC 管理 => 手动管理 (你负责调用 CFRelease 或 相关函数来放弃对象所有权） | +1               |
| __bridge_transfer (或 CFBridgingRelease) | CF => F  | 手动管理 => ARC 管理 (ARC 负责放弃对象的所有权)              | +1 再 -1，不改变 |

- **__bridge**（常用）：不改变对象的内存管理权所有者。

本来由`ARC`管理的`Foundation`对象，转换成`Core Foundation`对象后继续由`ARC`管理； 本来由开发者手动管理的`Core Foundation`对象，转换成`Foundation`对象后继续由开发者手动管理。 下面以`NSMutableArray`对象和`CFMutableArrayRef`对象为例：

```objc
    // 本来由 ARC 管理
    NSMutableArray *mArray = [[NSMutableArray alloc] init];
    // 转换后继续由 ARC 管理            
    CFMutableArrayRef cfMArray = (__bridge CFMutableArrayRef)(mArray); 

    // 本来由开发者手动管理
    CFMutableArrayRef cfMArray = CFArrayCreateMutable(kCFAllocatorDefault, 0, NULL); 
    // 转换后继续由开发者手动管理
    NSMutableArray *mArray = (__bridge NSMutableArray *)(cfMArray); 
    ...
    // 在不需要该对象时需要手动释放
    CFRelease(cfMArray); 
复制代码
```

`__bridge`的安全性与赋值给`__unsafe_unretained`修饰符相近甚至更低，如果使用不当没有注意对象的释放，就会因悬垂指针而导致`Crash`。

`__bridge`转换后不改变对象的引用计数，比如我们将`id`类型转换为`void *`类型，我们在使用`void *`之前该对象被销毁了，那么我们再使用`void *`访问该对象肯定会`Crash`。所以`void *`指针建议立即使用，如果我们要保存这个`void *`指针留着以后使用，那么建议使用`__bridge_retain`。

而在使用`__bridge`将`void *`类型转换为`id`类型时，一定要注意此时对象的内存管理还是由开发者手动管理，记得在不需要对象时进行释放，否则内存泄漏！

以下给出几个 “使用`__bridge`将`void *`类型转换为`id`类型” 的示例代码，要注意转换后还是由开发者手动管理内存，所以即使离开作用域，该对象还保存在内存中。

```objc
    // 使用 __strong
    CFMutableArrayRef cfMArray = CFArrayCreateMutable(kCFAllocatorDefault, 0, NULL);
    NSMutableArray *mArray = (__bridge NSMutableArray *)(cfMArray);
    
    NSLog(@"%ld", CFGetRetainCount(cfMArray));     // 2, 因为 mArray 是 __strong, 所以增加引用计数
    NSLog(@"%ld", _objc_rootRetainCount(mArray));  // 1, 但是使用 _objc_rootRetainCount 打印出来是 1 ？
     
    // 在不需要该对象时进行释放
    CFRelease(cfMArray); 
    NSLog(@"%ld", CFGetRetainCount(cfMArray));     // 1, 因为 __strong 作用域还没结束，还有强指针引用着
    NSLog(@"%ld", _objc_rootRetainCount(mArray));  // 1
    
   // 在 __strong 作用域结束前，还可以访问该对象
   // 等 __strong 作用域结束，该对象就会销毁，再访问就会崩溃
复制代码
    // 使用 __strong
    CFMutableArrayRef cfMArray;
    {
        cfMArray = CFArrayCreateMutable(kCFAllocatorDefault, 0, NULL);
        NSMutableArray *mArray = (__bridge NSMutableArray *)(cfMArray);
        NSLog(@"%ld", CFGetRetainCount(cfMArray));  // 2, 因为 mArray 是 __strong, 所以增加引用计数
    }
    NSLog(@"%ld", CFGetRetainCount(cfMArray));      // 1, __strong 作用域结束
    CFRelease(cfMArray); // 释放对象，否则内存泄漏
    // 可以使用 CFShow 函数打印 CF 对象
    CFShow(cfMArray);    // 再次访问就会崩溃
复制代码
    // 使用 __weak
    CFMutableArrayRef cfMArray = CFArrayCreateMutable(kCFAllocatorDefault, 0, NULL);
    NSMutableArray __weak *mArray = (__bridge NSMutableArray *)(cfMArray);
    
    NSLog(@"%ld", CFGetRetainCount(cfMArray));     // 1, 因为 mArray 是 __weak, 所以不增加引用计数
    NSLog(@"%ld", _objc_rootRetainCount(mArray));  // 1
    
    /*
     * 使用 mArray
     */
    
    // 在不需要该对象时进行释放
    CFRelease(cfMArray);
    
    NSLog(@"%@",mArray);  // nil, 这就是使用 __weak 的好处，在指向的对象被销毁的时候会自动置指针为 nil，再次访问也不会崩溃
复制代码
    // 使用 __weak
    NSMutableArray __weak *mArray;
    {
        CFMutableArrayRef cfMArray = CFArrayCreateMutable(kCFAllocatorDefault, 0, NULL);
        mArray = (__bridge NSMutableArray *)(cfMArray);
        NSLog(@"%ld", CFGetRetainCount(cfMArray));  // 1, 因为 mArray 是 __weak, 所以不增加引用计数
    }
    CFMutableArrayRef cfMArray =  (__bridge CFMutableArrayRef)(mArray);
    NSLog(@"%ld", CFGetRetainCount(cfMArray)); // 1, 可见即使出了作用域，对象也还没释放，因为内存管理权在我们
    
    CFRelease(cfMArray); // 释放对象，否则内存泄漏
复制代码
```

- **__bridge_retained**：用在`Foundation`对象转换成`Core Foundation`对象时，进行`ARC`内存管理权的剥夺。

本来由`ARC`管理的`Foundation`对象，转换成`Core Foundation`对象后，`ARC`不再继续管理该对象，需要由开发者自己手动释放该对象，否则会发生内存泄漏。

```objc
    // 本来由 ARC 管理
    NSMutableArray *mArray = [[NSMutableArray alloc] init];       
    // 转换后由开发者手动管理              
    CFMutableArrayRef cfMArray = (__bridge_retained CFMutableArrayRef)(mArray); 
//    CFMutableArrayRef cfMArray = (CFMutableArrayRef)CFBridgingRetain(mArray); // 另一种等效写法
    ...
    CFRelease(cfMArray);  // 在不需要该对象的时候记得手动释放
复制代码
```

`__bridge_retained`顾名思义会对对象`retain`，使转换赋值的变量也持有该对象，对象的引用计数 +1。由于转换后由开发者进行手动管理，所以再不需要该对象的时候记得调用`CFRelease`释放对象，否则内存泄漏。

```objc
    id obj = [[NSObject alloc] init];
    void *p = (__bridge_retained void *)(obj);
复制代码
```

以上代码如果在`MRC`下相当于：

```objc
    id obj = [[NSObject alloc] init];

    void *p = obj;
    [(id)p retain];
复制代码
```

**查看引用计数**

在`ARC`下，我们可以使用`_objc_rootRetainCount`函数查看对象的引用计数（该函数有时候不准确）。对于`Core Foundation`对象，我们可以使用`CFGetRetainCount`函数查看引用计数。

```objc
uintptr_t _objc_rootRetainCount(id obj);
复制代码
```

打印上面代码的`obj`对象的引用计数，发现其引用计数确实增加。

```objc
    id obj = [[NSObject alloc] init];
    NSLog(@"%ld", _objc_rootRetainCount(obj)); // 1
    void *p = (__bridge_retained void *)(obj);
    NSLog(@"%ld", _objc_rootRetainCount(obj)); // 2
    NSLog(@"%ld", CFGetRetainCount(p));        // 2
复制代码
```

以下给出几个示例代码：

```objc
    CFMutableArrayRef cfMArray;
    {
        NSMutableArray *mArray = [[NSMutableArray alloc] init];
        cfMArray = (__bridge_retained CFMutableArrayRef)(mArray);
        NSLog(@"%ld", CFGetRetainCount(cfMArray)); // 2, 因为 mArray 是 __strong, 而且使用了 __bridge_retained
    }
    NSLog(@"%ld", CFGetRetainCount(cfMArray));     // 1, __strong 作用域结束
    CFRelease(cfMArray); // 在不需要使用的时候释放，防止内存泄漏
复制代码
```

如果将上面的代码由`__bridge_retained`改为使用`__bridge`会怎样？

```objc
    CFMutableArrayRef cfMArray;
    {
        NSMutableArray *mArray = [[NSMutableArray alloc] init];
        cfMArray = (__bridge CFMutableArrayRef)(mArray);
        NSLog(@"%ld", CFGetRetainCount(cfMArray)); // 1, 因为 mArray 是 __strong, 且使用 __bridge 还是由 ARC 管理，不增加引用计数
    }
    NSLog(@"%ld", CFGetRetainCount(cfMArray)); // 程序崩溃，因为对象已销毁
复制代码
```

- **__bridge_transfer**：用在`Core Foundation`对象转换成`Foundation`对象时，进行内存管理权的移交。

本来由开发者手动管理的`Core Foundation`对象，转换成`Foundation`对象后，将内存管理权移交给`ARC`，开发者不用再关心对象的释放问题，不用担心内存泄漏。

```objc
    // 本来由开发者手动管理
    CFMutableArrayRef cfMArray = CFArrayCreateMutable(kCFAllocatorDefault, 0, NULL);
    // 转换后由 ARC 管理
    NSMutableArray *mArray = (__bridge_transfer NSMutableArray *)(cfMArray);
//    NSMutableArray *mArray = CFBridgingRelease(cfMArray); // 另一种等效写法
复制代码
```

`__bridge_transfer`作用如其名，移交内存管理权。它的实现跟`__bridge_retained`相反，会`release`被转换的变量持有的对象，但同时它在赋值给转换的变量时会对对象进行`retain`，所以引用计数不变。也就是说，对于`Core Foundation`引用计数语义而言，对象是释放的，但是`ARC`保留了对它的引用。

```objc
    id obj = (__bridge_transfer void *)(p);
复制代码
```

以上代码如果在`MRC`下相当于：

```objc
    id obj = (id)p;
    [obj retain];
    [(id)p release];
复制代码
```

下面也给出一个示例代码：

```objc
    CFMutableArrayRef cfMArray;
    {
        cfMArray = CFArrayCreateMutable(kCFAllocatorDefault, 0, NULL);
        NSMutableArray *mArray = (__bridge_transfer NSMutableArray *)(cfMArray);
        NSLog(@"%ld", CFGetRetainCount(cfMArray));    // 1, 因为 cfMArray 指针指向的对象存在，所以仍然可以通过该指针访问
        NSLog(@"%ld", _objc_rootRetainCount(mArray)); // 1, mArray 为 __strong
    }
    // __strong 作用域结束，ARC 对该对象进行了释放
    NSLog(@"%ld", CFGetRetainCount(cfMArray)); // 再次访问就会崩溃
复制代码
```

如果将上面的代码由`__bridge_transfer`改为使用`__bridge`会怎样？ 其实在`__bridge`讲解中已经给出了示例代码，如果不释放就会造成内存泄漏。

以上提到了可以替代`__bridge_retained`和`__bridge_transfer`的两个函数：`CFBridgingRetain`和`CFBridgingRelease`，下面我们来看一下函数实现：

```objc
/* Foundation - NSObject.h */
#if __has_feature(objc_arc)  // ARC

// After using a CFBridgingRetain on an NSObject, the caller must take responsibility for calling CFRelease at an appropriate time.
NS_INLINE CF_RETURNS_RETAINED CFTypeRef _Nullable CFBridgingRetain(id _Nullable X) {
    return (__bridge_retained CFTypeRef)X;
}

NS_INLINE id _Nullable CFBridgingRelease(CFTypeRef CF_CONSUMED _Nullable X) {
    return (__bridge_transfer id)X;
}

#else // MRC

// This function is intended for use while converting to ARC mode only.
NS_INLINE CF_RETURNS_RETAINED CFTypeRef _Nullable CFBridgingRetain(id _Nullable X) {
    return X ? CFRetain((CFTypeRef)X) : NULL;
}

// Casts a CoreFoundation object to an Objective-C object, transferring ownership to ARC (ie. no need to CFRelease to balance a prior +1 CFRetain count). NS_RETURNS_RETAINED is used to indicate that the Objective-C object returned has +1 retain count.  So the object is 'released' as far as CoreFoundation reference counting semantics are concerned, but retained (and in need of releasing) in the view of ARC. This function is intended for use while converting to ARC mode only.
NS_INLINE id _Nullable CFBridgingRelease(CFTypeRef CF_CONSUMED _Nullable X) NS_RETURNS_RETAINED {
    return [(id)CFMakeCollectable(X) autorelease];
}

#endif
复制代码
```

可以看到在`ARC`下，这两个函数就是使用了`__bridge_retained`和`__bridge_transfer`。

> **小结：** 在`ARC`下，必须恰当使用`Toll-Free Bridging`（`桥接`）在`Foundation`对象和`Core Foundation`对象之间进行类型转换，否则可能会导致内存泄漏。

**建议：**

> - 将`Foundation`对象转为`Core Foundation`对象时，如果我们立即使用该`Core Foundation`对象，使用`__bridge`；如果我们想保存着以后使用，使用`__bridge_retained`，但是要记得在使用完调用`CFRelease`释放对象。
> - 将`Core Foundation`对象转为`Foundation`对象时，使用`__bridge_transfer`。

### 编译器处理从 Cocoa 方法返回的 CF 对象

编译器知道返回`Core Foundation`对象的`Objective-C`方法遵循历史 Cocoa 命名约定。例如，编译器知道，在`iOS`中，`UIColor`的`CGColor`方法返回的`CGColor`并不持有（因为方法名不是以`alloc/new/copy/mutableCopy`开头）。所以你仍然必须使用适当的类型转换，如以下示例所示：

```objc
    NSMutableArray *colors = [NSMutableArray arrayWithObject:(id)[[UIColor darkGrayColor] CGColor]];
    [colors addObject:(id)[[UIColor lightGrayColor] CGColor]];
复制代码
```

否则编译警告：

```objc
    NSMutableArray *colors = [NSMutableArray arrayWithObject:[[UIColor darkGrayColor] CGColor]];
    // Incompatible pointer types sending 'CGColorRef _Nonnull' (aka 'struct CGColor *') to parameter of type 'id _Nonnull'
    // 不兼容的指针类型，将 CGColorRef（又称 struct CGColor *）作为 id 类型参数传入
复制代码
```

### 使用桥接转换函数参数

当在函数调用中在`Objective-C`和`Core Foundation`对象之间进行转换时，需要告诉编译器参数的所有权语义。`Core Foundation`对象的所有权规则请参阅[《Memory Management Programming Guide for Core Foundation》](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFMemoryMgmt/CFMemoryMgmt.html#//apple_ref/doc/uid/10000127i)。`Objective-C`的所有权规则请参阅`《从 MRC 说起 —— 内存管理策略》`章节。

如下实例所示，`NSArray`对象作为`CGGradientCreateWithColors`函数的参数传入，它的所有权不需要传递给该函数，因此需要使用`__bridge`进行强制转换。

```objc
    NSArray *colors = <#An array of colors#>;
    CGGradientRef gradient = CGGradientCreateWithColors(colorSpace, (__bridge CFArrayRef)colors, locations);
复制代码
```

以下实例中使用了`Core Foundation`对象以及`Objective-C`和`Core Foundation`对象之间进行转换，同时需要注意`Core Foundation`对象的内存管理。

```objc
- (void)drawRect:(CGRect)rect {
    CGContextRef ctx = UIGraphicsGetCurrentContext();
    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceGray();
    CGFloat locations[2] = {0.0, 1.0};
    NSMutableArray *colors = [NSMutableArray arrayWithObject:(id)[[UIColor darkGrayColor] CGColor]];
    [colors addObject:(id)[[UIColor lightGrayColor] CGColor]];
    CGGradientRef gradient = CGGradientCreateWithColors(colorSpace, (__bridge CFArrayRef)colors, locations);
    CGColorSpaceRelease(colorSpace);  // Release owned Core Foundation object.
    CGPoint startPoint = CGPointMake(0.0, 0.0);
    CGPoint endPoint = CGPointMake(CGRectGetMaxX(self.bounds), CGRectGetMaxY(self.bounds));
    CGContextDrawLinearGradient(ctx, gradient, startPoint, endPoint,
                                kCGGradientDrawsBeforeStartLocation | kCGGradientDrawsAfterEndLocation);
    CGGradientRelease(gradient);  // Release owned Core Foundation object.
}
复制代码
```

