

# 面向协议

# 引用类型&值类型



# .Type与.self、.Self:

### .Type

*元类型*是指任意类型的类型，包括类类型、结构体类型、枚举类型和协议类型。

类、结构体或枚举类型的元类型是相应的类型名紧跟 `.Type`。协议类型的元类型——并不是运行时遵循该协议的具体类型——是该协议名字紧跟 `.Protocol`。比如，类 `SomeClass` 的元类型就是 `SomeClass.Type`，协议 `SomeProtocol` 的元类型就是 `SomeProtocal.Protocol`。

### .Self

1. `Self`关键字用在类里， 作为函数返回值类型， 表示当前类，限定返回值跟方法调用者必须是同一类型。

2. Self可以用于协议(protocol)中限制相关的类型，比如**SomeProtocol**协议中 `func test() -> Self` 方法中返回`Self`，也就是说**Person**类继承了**SomeProtocol**协议必须返回Self(返回值必须是Person类型)。


   在定义协议的时候`Self`用的频率很高，而且当你**用错**`Self`的时候编译器会这样提示`'Self' is only available in a protocol or as the result of a method in a class

如：

`注意：`
1.test() 方法中不能直接返回Person(),否则会报类型不匹配错误；

原因： 主要是考虑，如果实现这个协议的类，有子类，即也就是有继承关系，那么直接返回父类将会报错,因为我们肯定希望调用子类就返回子类。

![image](./image/Swift-0.png)

