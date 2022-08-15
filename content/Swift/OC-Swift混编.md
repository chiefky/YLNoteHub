# OC-Swift 混编

## 1、在Swift中使用OC

### 1.1 在Swift使用OC类

> 只需要将OC类的头文件在bridge文件中import即可

### 1.2 在Swift使用OC方法/属性

1. 作为普通函数调用
2. 运行时使用(`performSelector:`)

## 2、在 OC中使用Swift

### 2.1 在OC中使用Swift类

> 在OC类中import “XXX-Swift.h”头文件

### 2.2 在OC中使用Swift方法/属性

> 有两种方法：
>
> 1. 每个方法、属性前家`@objc`关键字；
>
> 2. 类前面添加`@objcMembers`关键字（作用：对类和子类、扩展和子类扩展重新启用@objc推断）
>
>    <font color='orange'>但是，我们要尽量避免使用 @objcMembers，因为过多的 @objc 修饰，会增加 App 包的尺寸并影响性能。</font>
>
> > 附：
> >
> > **@objc的作用：**
> >
> > 1. @objc修饰符的根本目的是用来暴露接口给 Objective-C 的运行时（类、协议、属性和方法等）
> >
> > > 需要注意的是：<font color='red'>暴露给 Objective-C 的类必须是 NSObject 的子类</font>(因为纯swift类在oc中无法使用)，且该类、属性或方法必须使用 @objc 修饰，这样才能在 Objective-C 访问该类、属性或方法。
> >
> > 2.可以修改 Swift 接口暴露到 Objective-C 后的名字。
> >
> > ```swift
> > @objc(zhangsan)
> > class Белка: NSObject {
> >     @objc(name)
> >     var цвет: Цвет = .Красный
> > 
> >     @objc(hideNuts:inTree:)
> >     func прячьОрехи(количество: Int, вДереве дерево: Дерево) { }
> > }
> > ```
> >
> > 注意：添加@objc修饰符并不意味着这个方法或者属性会采用 Objective-C 的方式变成动态派发，Swift 依然可能会将其优化为静态调用。

### 2.3 如何保证 Swift 类中的 API 在swift类中 也是运行时调用的？

#### 使用`dynamic` 关键字

从功能上讲：
通过 @objc 或 @objcMembers，我们可以将 Swift 类暴露给 Objective-C，但是这样不能保证 Swift 类中的 API 是运行时调用的，因为 Swift 会做静态优化。

@objc dynamic 可以保证被修饰的 Swift API 是运行时被调用的，这样一来，我们就可以使用 Objective-C 的 KVO 或 Runtime System 的 method_exchangeImplementations 方法。

```swift
// 使用 KVO 监听 no 的变化
class SwiftStudent: ObjectiveCPerson {
    override var name: String! {
        set {
            super.name = newValue
        }
        get {
            return super.name
        }
    }
    @objc dynamic var no = "001"
}

override func observeValue(forKeyPath keyPath: String?, of object: Any?, change: [NSKeyValueChangeKey : Any]?, context: UnsafeMutableRawPointer?) {
    guard let newValue = change?[.newKey], let oldValue = change?[.oldKey] else {
        return
    }
    print("\n     New value: \(newValue)")
    print("     Old value: \(oldValue)")
}
```

# M、OC类不能继承swift类（即使是继承自NSObject的swift类）

> 您不能将SWIFT类划分为子类(即使它是NSObject的子类并且可用于Objective-C运行时)，因为OBJ-C中故意设置了限制，以阻止在OBJ-C代码中划分SWIFT类的子类。
>
> 我认为这一限制的原因是，SWIFT包含不能在OBJ-C中使用的功能，因此子类将受到限制，并且在实现不能跨入OBJ-C的方法时会得到未定义的行为。
>
> 如果苹果允许Obj-C->Swift->Obj-C子类化，那么它将是有限的。有些方法在SWIFT中的作用不同于它们在Obj-C选择器中的作用，理论上，您可以在子类中声明冲突的方法，这些方法将具有不同的操作，具体取决于您是在使用SWIFT还是在Obj-C中处理类。此外，SWIFT编译器无法超越SWIFT障碍，因此可能会进行优化，从而破坏您的Obj-C子类。

