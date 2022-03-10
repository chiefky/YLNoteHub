#  block内循环引用解决方案

## 1. 借助weak解决循环引用

```objective-c
- (void)testBlockLifeCircle_weak {
  __weak typeof(self) weakSelf = self;

  self.yl_VBlock = ^{
    NSLog(@"demoName = %@",[weakSelf demoName]);
  };
  self.yl_VBlock();
}	
```

## 2. 借助__block解决循环引用

```objective-c
- (void)testBlockLifeCircle_block {

  __block YLBlockRetainCycleViewController *tmpVC = self;
  self.yl_VBlock = ^{
    NSLog(@"demoName = %@",[tmpVC demoName]);
    tmpVC = nil; // 📢：1.必须将变量置为nil
  };
  self.yl_VBlock(); // 📢：2.必须调用block(如果不调不会，解决循环引用)
}
```



## 3. "将强引用对象作为block参数传入"解决循环引用

```objective-c
- (void)testBlockLifeCircle_Parameter {
  self.yl_PBlock = ^(YLBlockRetainCycleViewController *vc) {
    NSLog(@"demoName = %@",[vc demoName]);
  };
  self.yl_PBlock(self);
}

```

## 4. 借助NSProxy解决循环引用

* 方案四：使用NSProxy（其实是借助参数传递中间者+中间者弱持有self就可以解决，完全可以不用消息转发）（使用Proxy的原理是：1.添加了一个中间者Proxy；2.Proxy持有一个弱引用对象，也就是响应方法的目标对象；3. 借助消息转发机制将消息传递给目标对象）

  ```objective-c
  - (void)testBlockLifeCircle_Rroxy {
    YLProxy *proxy = [YLProxy proxyWithTarget:self];
    self.yl_PoxBlock = ^(YLProxy *pox) {
      NSLog(@"demoName = %@",[pox.target demoName]);
    };
    self.yl_PoxBlock(proxy);
  }
  ```

  