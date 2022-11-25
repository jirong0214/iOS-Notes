## RunLoop  维护事件循环来对 事件/消息 进行管理

1. 事件循环原理

- 没有消息时，休眠进入内核态，避免资源占用
- 有消息需要处理时，立即唤醒进入用户态处理消息
- 通过调用`mach_msg()`函数来将当前线程的控制权交给内核态/用户态

2. RunLoop应用：

- 保持App持续运行
- 事件响应、手势响应、界面刷新
- Timer 、PerformSelector、GCD、网络请求
- autoreleaseppool

### RunLoop底层数据结构 （CFRunLoopRef）

struct CFRunLoopRef:

- _pthread：`RunLoop`与线程是**一一对应关系**
- _commonModes：存储着 NSString 对象的集合（Mode 的名称）
- _commonModeItems：存储着被标记为通用模式的`Source0`/`Source1`/`Timer`/`Observer`
- _currentMode：`RunLoop`当前的运行模式
- _modes：存储着`RunLoop`所有的 Mode（`CFRunLoopModeRef`）模式

```objc
struct __CFRunLoopMode {
    CFStringRef _name;             // mode 类型，如：NSDefaultRunLoopMode
    CFMutableSetRef _sources0;     // CFRunLoopSourceRef
    CFMutableSetRef _sources1;     // CFRunLoopSourceRef
    CFMutableArrayRef _observers;  // CFRunLoopObserverRef
    CFMutableArrayRef _timers;     // CFRunLoopTimerRef
    ...
}
```

**为什么Runloop要分mode？**

可以根据mode去处理对应的事件: 比如`NSDefaultRunLoopMode`默认模式和`UITrackingRunLoopMode`滚动模式，滚动屏幕的时候就会切换到滚动模式，就不用去处理默认模式下的事件了，保证了 UITableView 等的滚动顺畅。

**事件源**：

| Input Sources | 区别                                                         |
| ------------- | ------------------------------------------------------------ |
| Source0       | 需要手动唤醒线程：添加`Source0`到`RunLoop`并不会主动唤醒线程，需要手动唤醒） ① 触摸事件处理 ② `performSelector:onThread:` |
| Source1       | 具备唤醒线程的能力 ① 基于 Port 的线程间通信 ② 系统事件捕捉：系统事件捕捉是由`Source1`来处理，然后再交给`Source0`处理 |

**Timers:**

使用NSTimer进行定时任务时需要将其加入到RunLoop中，才能定时运行；另外，使用`performSelector:withObject:afterDelay:`方法时，也会创建`timer`并添加到`RunLoop`中。

**Observers**:

观察Runloop的活动状态的观察者

### 主线程 RunLoop 的调用过程

**iOS 程序持续运行的原因**： UIApplicationMain中调用了 **CFRunLoopRunSpecific** 函数，使主线程的RunLoop运行起来。

```objc
SInt32 CFRunLoopRunSpecific(CFRunLoopRef rl, CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) { 
    // 通知 Observers：即将进入 RunLoop
    __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);
    // RunLoop 具体要做的事情
	result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode);
    // 通知 Observers：即将退出 RunLoop
    __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);

    return result;
}
```

**事件循环的实现机制**: __CFRunLoopRun函数：

```objc
__CFRunLoopDoObservers // 通知Observers接下来要做什么
__CFRunLoopDoBlocks // 处理Blocks
__CFRunLoopDoSources0 // 处理Sources0
__CFRunLoopDoSources1 // 处理Sources1
__CFRunLoopDoTimers // 处理Timers
dispatch_async(dispatch_get_main_queue(), ^{ }); // 处理 GCD 相关：
__CFRunLoopSetSleeping/__CFRunLoopUnsetSleeping：// 休眠等待/结束休眠
__CFRunLoopServiceMachPort -> mach-msg()：// 转移当前线程的控制权
```

**RunLoop 休眠的实现原理**:

调用`mach_msg()`函数来转移当前线程的控制权给内核态进入休眠, 此时不占用CPU资源；有消息需要处理时，调用`mach_msg()`回到用户态（唤醒线程）处理消息。

`RunLoop`与简单的`do...while`循环的区别：

- RunLoop中当无消息进入休眠的时候，不会占用CPU资源
- 简单的do-while循环是一直占用CPU资源的

### Runloop 与线程

CFRunLoopGetCurrent();  *// 获取当前线程的 RunLoop 对象*

如何获取的？以当前thread为key，从哈希表中查询runloop对象；如果不存在，则创建runloop对象并返回；(以及更新哈希表)

使用runloop 进行子线程的保活:

- 获取/创建当前子线程的RunLoop；

- 向该RunLoop添加一个port来维持runloop的事件循环；（port或source或timer不能全为空，否则runloop立即停止）；

- 启动该runloop：

  - [[NSRunLoop currentRunLoop] run]: 该线程会一直存在，不会被销毁，因为run方法会一直调用*runMode:beforeDate: 方法，以运行在 NSDefaultRunLoopMode 模式下

  - ```objc
    while (weakSelf && weakSelf.thread) {
      [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
    }
    ```

    这样可以只让self和thread存在的情况下才执行runMode

### RunLoop 与 NSTimer

NSTimer 的定时任务也是RunLoop来管理的，如果要在子线程上使用NSTimer，则必须开启子线程的RunLoop，否则定时器无法生效；

**解决tableView滑动时NSTimer失效的问题:** 

因为滑动`tableview`/`scrollview`的时候`RunLoop`会切换到`UITrackingRunLoopMode`模式下，所以不能使用`NSDefaultRunLoopMode`，可以使用`NSRunLoopCommonModes`，这样意味着在UITracking和Default模式下该Timer都可以执行；

```objc
NSTimer *timer = [NSTimer timerWithTimeInterval:1.0 repeats:YES block:^(NSTimer * _Nonnull timer) {
  NSLog(@"123");
}];
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes]; // 注意这里要用NSRunLoopCommonModes
```

**解决NSTimer和CADisplayLink不准时的问题：**

比如`NSTimer`每1.0秒就会执行一次任务，`Runloop`每进行一次循环，就会看一下`NSTimer`的时间是否达到1.0秒，是的话就执行任务；(参考: A timer is not a real-time mechanism; it fires only when one of the run loop modes to which the timer has been added is running and able to check if the timer’s firing time has passed.) 但是RunLoop每次循环的时间是不固定的，导致到调用NSTimer的任务时会有一定的时间偏差；

CADisplayLink 是用于同步屏幕刷新的定时器，如果任务繁重的话，会出现丢帧现象的, 因此也不准时

解决方法：使用GCD创建定时器，不依赖于RunLoop；

```objc
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(10.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
    // TO DO METHOD CALL.
});
```

**解决NSTimer循环引用的问题:**

注意，使用weakSelf是无用的，因为timer与block不同，不会根据weakSelf的修饰符来决定是否强引用；

解决方法1：使用基于block的NSTimer方法，让Timer不引用self

解决方法2：在适当的地方调用invalidate（），例如在页面willDisappear时；

方法3：使用NSProxy; （待掌握）