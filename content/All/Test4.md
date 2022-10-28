**一面**



1、项目经历：介绍一下自己参与过的项目中的亮点？

2、消息转发的过程？一般用来干嘛？

3、Runloop有几种state？和子线程的关系？Runloop常见的使用场景？

4、Autoreleasepool的数据结构是什么？什么时候释放？

Autoreleasepool没有具体的结构，底层实现是一个由AutoreleasepoolPage组成的双向链表，完成autorelease对象的添加和释放；

系统添加的全局Autoreleasepool释放由Runloop 控制，在runloop beforewaiting时和exit时都会触发释放。

5、Atomic的属性是否安全？比如数组？

6、For in 遍历数组时 内部删除元素会怎样？

7、iOS常见的锁有哪几种？性能怎么样？

8、@sychronized 是否可以嵌套？为什么？

9、介绍下iOS应用的启动过程？rebase流程是干嘛的？优化应用启动速度的方式有哪些？

10、二进制重排优化了解吗？能简单说下原理吗？

11、优化页面显示性能的方式有了解吗？

12、多线程问题：实现A、B任务结束执行C任务方法有几种？想到几种说几种？

13、pod install、pod update 、podfile、podfile.lock 与 pod repo update 的联系与作用

14、算法：使用两个队列实现栈？



**二面**



1、什么是懒加载类？什么是非懒加载类？

2、load方法是在什么时候调用的？load方法内部可以调用其他类的方法吗？

3、load 方法可以hook吗？怎么hook？

4、怎么查找全局的内存泄露？

5、引起内存泄露的情况有哪些？CoreFoundation 转 Foundation 会产生吗？toll_free bridge 了解吗？可以介绍下？

6、tcp 的拥塞控制了解吗？有哪些常见的算法？quick 协议有了解吗？

7、打点SDK能否实现 页面滑出屏幕 自动打点？

9、线程通信的方式有哪些？

13、算法：两个字符串最长的相同子串？动态规划了解吗？