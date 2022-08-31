## 1. 以下代码将会怎样执行？换成注释中的代码又会怎样执行？

```objective-c
- (void)testStackBlock_copy {
    int num = 8;
    void(^__weak weakBlock)(void) = nil;
    {
        void(^__weak weakBlock1)(void) = ^{
            NSLog(@"num = %d",num);
        };  
        weakBlock = weakBlock1;// [weakBlock1 copy];
    }
    
    weakBlock();
    
}
```

> 答： 
>
> 两种情况分析：
>
> ​    1." weakBlock = weakBlock1; "
>
> ​     两个block都是stack block，都存储在栈区；栈区空间是方法体空间，中间的'{}'表示匿名作用域，（C语言中介绍）栈存储期间匿名作用域的变量超出匿名作用域不一定立即释放；
>
> ​     2. "weakBlock = [weakBlock1 copy];"
>
> ​     经过copy后 weakBlock指向的是堆区的空间,此空间在给weakBlock赋值前被ARC插入的release释放掉，所以weakBlock为null;
>
> ​     超出"}"后调用weakBlock(),相当于访问了野指针所以会崩溃。

## 2. 什么情况下系统不会对block执行copy操作？

答：stack block作为函数参数时，系统无法判断是否需要进行copy，此时不会主动进行copy操作；

## 3. block的参数与返回值用途

```objective-c
typedef ActionHandler: BOOL(^)()
```

A-->B

## 3. block的原理是怎样的？本质是什么？

## 4.`__block`的作用是什么？有什么使用注意点？

## 5.block的属性修饰词为什么是copy？使用block有哪些使用注意？

## 6.block一旦没有进行copy操作，就不会在堆上

## 7.block在修改NSMutableArray，需不需要添加__block？



\
