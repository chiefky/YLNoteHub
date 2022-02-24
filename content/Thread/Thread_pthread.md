# 1.pthread

- 全称 POSIX Thread，POSIX（Portable Operating System Interface）表示可移植操作系统接口；
- 一套用 C 语言写的通用的多线程 API；
- 适用于 Unix / Linux / Windows 等系统；
- 跨平台/可移植；
- 使用难度大、使用频率低；
- 线程生命周期由程序员管理；
- 现在 iOS 中用到 pthread 的多数情况是使用 pthread_mutex 互斥锁，性能较高。

## 1.2 pthread 的简单使用

```objectivec
#import <pthread.h>

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
    //创建子线程
    pthread_t pthread; //线程编号
    /*
     参数一：线程编号的地址: 要创建线程的结构体指针，通常开发的时候，如果遇到 C 语言的结构体，类型后缀 _t / Ref 结尾
 同时不需要 *
     参数二：线程的属性: nil(空对象 - OC 使用的) / NULL(空地址，0 C 使用的)
     参数三：线程要执行的函数 void * （*）（void *）: 
     线程要执行的函数地址,
 void *: 返回类型，表示指向任意对象的指针，和 OC 中的 id 类似
(*): 函数名
 (void *): 参数类型，void *
     参数四：函数的参数，参数类型：void *: 传递给第三个参数(函数)的参数
     返回值：0代表成功，非0代表失败
     pthread_create(pthread_t  _Nullable *restrict _Nonnull,
                    const pthread_attr_t *restrict _Nullable,
                    void * _Nullable (* _Nonnull)(void * _Nullable),
                    void *restrict _Nullable)
     */
    int result = pthread_create(&pthread, NULL, demo, NULL);
    ifz (result == 0) {
        NSLog(@"成功");
    } else {
        NSLog(@"失败");
    }
}

void *demo(void *param) {
    NSLog(@"hello,%@",[NSThread currentThread]);
    return NULL;
}
```

#  2. `__bridge`、` __bridge_retained`、`__bridge_transfer`

**C**和**OC**的桥接



其中涉及C与OC的桥接，有以下几点说明:

- __bridge只做类型转换，但是不修改对象(内存)管理权
- __bridge_retained（也可以使用CFBridgingRetain）将Objective-C的对象转换为 Core Foundation 的对象，同时将对象(内存)的管理权交给我们，后续需要使用 CFRelease或者相关方法来释放对象
- __bridge_transfer（也可以使用CFBridgingRelease）将Core Foundation的对象 转换为Objective-C的对象，同时将对象(内存)的管理权交给ARC。