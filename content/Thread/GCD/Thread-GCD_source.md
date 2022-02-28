**11**、**dispatch_source**



- 通过dispatch_source_create创建一个对象:

​    dispatch_source_t dispatch_source_create(dispatch_source_type_t type, uintptr_t handle, unsigned long mask, dispatch_queue_t queue);



   第一个参数是一个dispatch_source 类型，提供的类型如下：

- - DISPATCH_SOURCE_TYPE_DATA_ADD
  - DISPATCH_SOURCE_TYPE_DATA_OR
  - DISPATCH_SOURCE_TYPE_MACH_RECV
  - DISPATCH_SOURCE_TYPE_MACH_SEND
  - DISPATCH_SOURCE_TYPE_PROC
  - DISPATCH_SOURCE_TYPE_READ
  - DISPATCH_SOURCE_TYPE_SIGNAL
  - DISPATCH_SOURCE_TYPE_TIMER
  - DISPATCH_SOURCE_TYPE_VNODE
  - DISPATCH_SOURCE_TYPE_WRITE
  - DISPATCH_SOURCE_TYPE_MEMORYPRESSURE



   使用 dispatch_source 实现 timer 计时器功能：

   dispatch_queue_t queue = dispatch_queue_create("com.tangh.test", DISPATCH_QUEUE_CONCURRENT);

   

dispatch_source_t source = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);

dispatch_source_set_timer(self.source, dispatch_walltime(NULL, 0), 5ull * NSEC_PER_SEC, 0);



dispatch_source_set_event_handler(self.source, ^{

   [self testSelector];

});



dispatch_source_set_cancel_handler(self.source, ^{

   NSLog(@"");

});



 dispatch_resume(source);