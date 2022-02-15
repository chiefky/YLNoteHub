# Runloop 与NSPort

参考：

1. [**iOS**开发·**RunLoop**源码与用法完全解析**(**输入源，定时源，观察者，线程间通信，端口间通信，**NSPort**，**NSMessagePort**，**NSMachPort**，**NSPortMessage)**](https://www.jianshu.com/p/07313bc6fd24)
2. [**NSPort**线程间通信实例（**Xcode12.3**）](https://juejin.cn/post/6922283273012019213)



#### 1. 线程间通信的一种方式（另一种是performSelectorOnThread：）

核心思路：

1. 在主线程创建一个NSPort 的实例A 并添加主线程的NSRunLoop中。
2. 再创建一个线程 S 将A作为参数发送到线程中。
3. 线程 S 中创建一个本地NSPort 实例B，也添加到NSRunLoop中。
4. 从B向A发送消息后，将线程S的runloop运行。
5. 主线程收到线程S发送的消息。
6. 主线程向线程S发送消息。
7. 通过 handlePortMessage: 代理方法接受消息。

