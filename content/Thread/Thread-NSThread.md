# 1. NSThread 介绍

- 使用更加面向对象；
- 简单易用，可直接操作线程对象；
- 语言 OC，线程生命周期由程序员管理，偶尔使用。

### 初始化接口

加上iOS10新增的，现在有5种初始化方法，实例方法中非block初始化的需要调用start方法进行线程开启：

- \+ (void)detachNewThreadWithBlock:(void (^)(void))block; // iOS 10 新增
- \+ (void)detachNewThreadSelector:(SEL)selector toTarget:(id)target withObject:(nullable id)argument;
- \- (instancetype)initWithBlock:(void (^)(void))block; // iOS 10 新增
- \- (instancetype)initWithTarget:(id)target selector:(SEL)selector object:(nullable id)argument;
- \- (instancetype)init;



# 2. 线程优先级权限

在iOS中，目前线程优先级有以下几种权限：

```objective-c
typedef NS_ENUM(NSInteger, NSQualityOfService) {
  // 用户交互权限，属于最高等级，常被用于处理交互事件或者刷新UI，因为这些需要即时的
  NSQualityOfServiceUserInteractive = 0x21,
  
  // 为了可以进一步的后续操作，当用户发起请求结果需要被立即展示，比如当点了列表页某条信息后需要立即加载详情信息
  NSQualityOfServiceUserInitiated = 0x19,

  // 不需要马上就能得到结果，比如下载任务。当资源被限制后，此权限的任务将运行在节能模式下以提供更多资源给更高的优先级任务
  NSQualityOfServiceUtility = 0x11,

  // 后台权限，通常用户都不能意识到有任务正在进行，比如数据备份等。大多数处于节能模式下，需要把资源让出来给更高的优先级任务
  NSQualityOfServiceBackground = 0x09,

  // 默认权限，具体权限由系统根据实际情况来决定使用哪个等级权限，如果实际情况不太利于决定使用何种权限，则从UserInitiated和Utility之间选一个权限并使用
  NSQualityOfServiceDefault = -1

 }

```

关于`NSQualityOfServiceDefault`官方解释：<font color='red'>The priority level of this QoS falls between user-initiated and utility. This QoS is not intended to be used by developers to classify work. Work that has no QoS information assigned is treated as default, and the GCD global queue runs at this level.</font>

总结：

- NSThread 适用于业务场景不是很复杂的时候，属于比较轻量级的场景，同时由于对NSThread进行了相关拓展，所以开发中调用也极其方便。
- NSThread 缺点也很明显，不能设置线程之间的依赖关系，需要手动管理睡眠唤醒等。
