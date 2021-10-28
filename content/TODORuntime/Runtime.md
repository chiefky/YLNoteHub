##  1. 关联对象

## 2. 类的方法是如何被加载的



## 3 消息转发流程

怎么打开汇编查看流程,有什么好处?

答：菜单栏Debug->Debug Workflow->Always Show Disassembly（始终显示汇编代码）

cache` 在什么时候开始扩容 , 为什么? 

类方法存在哪里? 为什么要这么设计? 

方法慢速查找过程中的二分查找流程,请用伪代码实现 ？

类的结构里面为什么会有 `rw` 和 `ro` 以及 `rwe` ? 

`objc_msgSend` 为什么用汇编写 , `objc_msgSend` 是如何递归找到imp? 

一个类的类方法没有实现为什么可以调用 `NSObject` 同名对象方法

## 6. Method-Swizzing

## 面试题

1. iskindOfClass & isMemberOfClass 

   > // Objc 源码
   >
   > \+ (Class)class {
   >
   >   return self;
   >
   > }
   >
   > 
   >
   > \- (Class)class {
   >
   >   return object_getClass(self);
   >
   > }
   >
   > 
   >
   >  +(BOOL)isMemberOfClass:(Class)cls {
   >
   >   return self->ISA() == cls;
   >
   > }
   >
   > 
   >
   > -(BOOL)isMemberOfClass:(Class)cls {
   >
   >   return [self class] == cls;
   >
   > }
   >
   > 
   >
   > \+ (BOOL)isKindOfClass:(Class)cls {
   >
   >   for (Class tcls = self->ISA(); tcls; tcls = tcls->getSuperclass()) {
   >
   > ​    if (tcls == cls) return YES;
   >
   >   }
   >
   >   return NO;
   >
   > }
   >
   > 
   >
   > \- (BOOL)isKindOfClass:(Class)cls {
   >
   >   for (Class tcls = [self class]; tcls; tcls = tcls->getSuperclass()) {
   >
   > ​    if (tcls == cls) return YES;
   >
   >   }
   >
   >   return NO;
   >
   > }
   >
   > 

   ```objective-c
   //-----使用 iskindOfClass & isMemberOfClass 类方法
   BOOL re1 = [(id)[NSObject class] isKindOfClass:[NSObject class]];       // 
   BOOL re2 = [(id)[NSObject class] isMemberOfClass:[NSObject class]];     //
   BOOL re3 = [(id)[LGPerson class] isKindOfClass:[LGPerson class]];       //
   BOOL re4 = [(id)[LGPerson class] isMemberOfClass:[LGPerson class]];     //
   NSLog(@" re1 :%hhd\n re2 :%hhd\n re3 :%hhd\n re4 :%hhd\n",re1,re2,re3,re4);
   
   ```

   ```objc
   //------iskindOfClass & isMemberOfClass 实例方法
   BOOL re5 = [(id)[NSObject alloc] isKindOfClass:[NSObject class]];       //
   BOOL re6 = [(id)[NSObject alloc] isMemberOfClass:[NSObject class]];     //
   BOOL re7 = [(id)[LGPerson alloc] isKindOfClass:[LGPerson class]];       //
   BOOL re8 = [(id)[LGPerson alloc] isMemberOfClass:[LGPerson class]];     //
   NSLog(@" re5 :%hhd\n re6 :%hhd\n re7 :%hhd\n re8 :%hhd\n",re5,re6,re7,re8);
   ```

   总结：

    isKindOfClass 比较过程：

    【+】类方法：元类（isa） --> 元类的父类  --> 根类（父类） --> nil（父类） 与 传入类的对比

    【-】实例方法：对象的类(isa) --> 父类 --> 根类 --> nil 与 传入类的对比

   

    isMemberOfClass  比较过程：

    【+】类方法： 类的元类(isa) 与 传入类 对比

    【-】实例方法：对象的类(isa) 与 传入类 对比

