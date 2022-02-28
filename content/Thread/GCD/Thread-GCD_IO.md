# **GCD** 文件读取：**Disapatch I/O**

 在读取较大的文件时，如果将文件分成合适的大小并使用 Global Dispatch Queue 并行读取的话，应该会比一般的读取速度快不少。 在 GCD 当中能实现这一功能的就是 Dispatch I/O 和 Dispatch Data。



- 异步串行读取文件

   🌰代码：





- 异步并行读取文件

   🌰代码：

   NSString *desktop = @"/Users/zq/Desktop/多线程_GCD-18-7-23-0/多线程_GCD-18-7-23-0/";

  NSString *path = [desktop stringByAppendingPathComponent:@"ViewController.m"];



  dispatch_fd_t fd = open(path.UTF8String, O_RDONLY);

   

  dispatch_queue_t queue = dispatch_queue_create("com.myThread.concurrent", DISPATCH_QUEUE_CONCURRENT);

  dispatch_io_t io = dispatch_io_create(DISPATCH_IO_RANDOM, fd, queue, ^(int error) {

​    close(fd);

  });

   

  dispatch_group_t group = dispatch_group_create();

  NSMutableData *totalData = [[NSMutableData alloc] initWithLength:fileSize];



   /**设置读取的文件大小*/

  off_t currentSize = 0;

  long long fileSize = [[NSFileManager defaultManager] attributesOfItemAtPath:path error:nil].fileSize;

  size_t offset = 1024*1024;



  for (; currentSize <= fileSize; currentSize += offset) {

​    dispatch_group_enter(group);



​    dispatch_io_read(io, currentSize, offset, queue, ^(bool done, dispatch_data_t _Nullable data, int error) {

​      if (error == 0) {

​        size_t len = dispatch_data_get_size(data);

​        if (len > 0) {

​          const void *bytes = NULL;

​          (void)dispatch_data_create_map(data, (const void **)&bytes, &len);

​          [totalData replaceBytesInRange:NSMakeRange(currentSize, len) withBytes:bytes length:len];

​        }

​      }



​      if (done) {

​        dispatch_group_leave(group);

​      }

​    });

  }



  dispatch_group_notify(group, queue, ^{

​    NSString *str = [[NSString alloc] initWithData:totalData encoding:NSUTF8StringEncoding];

​    NSLog(@"%@", str);

  });