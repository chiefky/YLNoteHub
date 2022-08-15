# 1. iOS 常用的计时器有哪几种

## 1.1 NSTimer

> 使用场景：重复执行某种操作

## 1.2 CADisplayLink

> 使用场景：从原理上可以看出，CADisplayLink适合做界面的不停重绘，比如视频播放的时候需要不停地获取下一帧用于界面渲染。

## 1.2 DispatchSourceTimer

> - 最小精度为纳秒,误差在50毫秒以内.
> - 比较常用的 dispatch_after方法并没有直接的cancel方法