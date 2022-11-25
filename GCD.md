## GCD

GCD自动管理线程的声明周期（创建、调度、销毁线程）， 使用GCD不需要编写线程管理代码；

在OBJC中，GCD是纯C语言API；

## GCD 的任务：

- ####  dispatch_block_t（常用）

- #### dispatch_function_t

### GCD 的使用步骤：

- 创建队列 （FIFO）
- 创建任务
- 任务添加进队列（并制定任务执行方式）

<img src="/Users/tianjirong/Library/Mobile Documents/com~apple~CloudDocs/笔记/assets/image-20221111150544913.png" alt="image-20221111150544913" style="zoom:33%;" />

GCD是如何使用线程的？

GCD 中，要执行队列中的任务时，会自动开启一个线程，当任务执行完，线程不会立刻销毁，而是**放到了线程池中**。如果接下来还要执行任务的话就从线程池中取出线程，这样**节省了创建线程所需要的时间**。但如果一段时间内没有执行任务的话，该线程就会被销毁，再执行任务就会创建新的线程。

### GCD 执行任务的方式

### 1. 同步：在执行完后返回（阻塞）

#### 	● dispatch_sync: 使用block；

#### 	● dispatch_sync_f：使用C语言 function;

### 2. 异步：提交一个任务到队列后，直接返回（不阻塞）

#### 	● dispatch_async

#### 	● dispatch_async_f

### 同步和异步的区别

- **同步**：必须等待当前语句执行完毕，才会执行下一条语句（阻塞）；
  在`当前`线程中执行任务，`不具备`开启新线程的能力。
- **异步**：不用等待当前语句执行完毕，就可以执行下一条语句（不会阻塞）；
  在`新的`线程中执行任务，`具备`开启新线程的能力。（具备开启新线程的能力，不代表一定能开启新线程。如在主队列异步执行，不会开启新线程，因为主队列的任务在主线程上执行）

## GCD队列

除了主队列在主线程上执行以外，系统无法保证它使用哪个线程来执行任务。

### 队列类型

<img src="/Users/tianjirong/Library/Mobile Documents/com~apple~CloudDocs/笔记/assets/image-20221111154932429.png" alt="image-20221111154932429" style="zoom:33%;" />

**串行队列**（`DISPATCH _QUEUE _SERIAL`）：向该队列添加的任务只能一个接一个的执行；（**故GCD只创建1个线程就够了**）

**并发队列**（`DISPATCH _QUEUE _CONCURRENT`）：多个任务并发（同时）执行（**自动开启多个线程执行任务**）；常配合barrier一起使用；

#### 特殊队列：

- **主队列**（`dispatch_queue_main_t`）主队列的任务都在主线程上执行，主队列在程序一开始就被系统创建并与主线程关联。

- **全局并发队列 **（dispatch_get_global_queue）：可以指定服务质量（服务质量有助于确定队列执行的任务的优先级）。

  与普通并发队列相比: 系统管理，没有手动指定的label，不需要考虑内存管理，一般我们使用全局并发队列；

| 执行方式      | 并发队列                        | 手动创建的串行队列              | 主队列                          |
| ------------- | ------------------------------- | ------------------------------- | ------------------------------- |
| 同步（sync）  | `没有`开启新线程 `串行`执行任务 | `没有`开启新线程 `串行`执行任务 | `没有`开启新线程 `串行`执行任务 |
| 异步（async） | `有`开启新线程 `并发`执行任务   | `有`开启新线程 `串行`执行任务   | `没有`开启新线程 `串行`执行任务 |

sync(并发队列)：退化为串行执行；因为相当于限制了往队列中添加任务的速度（执行完才能添加）；

### 死锁问题：

死锁四大条件：

- **互斥**：某种资源一次只允许一个进程访问；
- **请求与保持**：一个进程本身占有资源，同时还有资源未得到满足，请求新资源的同时一直保持着现有资源
- **不可剥夺**：已获得的资源在未使用完之前，不能被剥夺，只能在使用完时由自己释放。
- **循环等待**：等待资源成环。

解决死锁办法：破坏这四大条件中的其一

### GCD中的死锁：

使用**dispatch_sync**向**当前串行队列中**添加同步任务，会导致循环等待；

解决方法：1 or 2 or 3

1. 改为使用dispatch_async添加异步任务
2. 切到另一个队列向该队列添加任务
3. 当前队列改为使用并发而不是串行队列



#### QoS相关用法:

告诉系统我们在进行什么样的工作，然后系统会通过合理的资源控制来最高效的执行任务代码，其中主要涉及到 CPU 调度的优先级、IO 优先级、任务运行在哪个线程以及运行的顺序等。

优先级从高到低：UserInteractive、UserInitiated、Default（默认）、Utility、Background

在使用全局并发队列时需要制定QoS优先级；

指定一个队列的QoS属性:

```objc
dispatch_queue_attr_t backgroundQueue_attr = dispatch_queue_attr_make_with_qos_class (DISPATCH_QUEUE_SERIAL, QOS_CLASS_BACKGROUND, -1);
dispatch_queue_t userInitiatedQueue = dispatch_queue_create("myqueue1", userInitiatedQueue_attr);
```

还可以使用dispatch_set_target_queue(queue1, targetQueue): 将targetQueue的属性设置给queue1；（包括QoS优先级）



#### DispatchBlock的相关用法:

1. **创建一个带有 QoS 的 block，指定 block 的优先级。**

```objc
dispatch_block_t BackGroundBlock = dispatch_block_create_with_qos_class(0, QOS_CLASS_BACKGROUND, -1, ^{
            NSLog(@"BackGroundBlock");
});
```

这样可以指定Block的优先级，（但是得是并发队列里才有用，因为串行队列都是单线程的）

#### 2. dispatch_block_notify:

```objc
dispatch_block_notify(observation_block, dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), notification_block); // 当observation_block执行完后，通知notification_block提交到global_queue;
```

#### 3. dispatch_block_wait:

```objc
dispatch_block_wait(block, DISPATCH_TIME_FOREVER); // 同步阻塞等待，直到指定的 block 执行完成或指定的超时时间结束为止才继续向下执行；可以设置一定的超时时间，设置DISPATCH_TIME_FOREVER是一直等待直到block执行完

```



#### 4. dispatch_block_cancel：

异步取消指定的 block，正在执行的 block 不会被取消；

####  5. dispatch_block_testcancel：

测试该 block 是否被取消；



## Dispatch Group 队列组：所有任务执行完成后有一个统一的回调

1. #### dispatch_group_create 创建一个group

2. #### dispatch_group_async: 在指定group和queue上，异步执行任务；（实际常用enter/leave手动控制一个任务的完成）

3. #### dispatch_group_notify: 当该group上的所有任务都执行完后，在指定的queue上执行回调

4. #### dispatch_group_wait: 同步等待该group上的所有任务都执行完，才继续往下执行；

**原理：**

**dispatch_group_async** 等价于：

```objc
dispatch_group_enter(group); // 进入队列组，执行此函数后，再添加的异步执行的block任务都会被group监听
    dispatch_async(queue, ^{
    dispatch_group_leave(group); // 完成1个任务
});
```

适用场景：启动多个网络请求，等全部返回之后再进行notify回调。



## Dispatch Once 一次性执行

#### dispatch_once:

```objc
static dispatch_once_t onceToken;
dispatch_once(&onceToken, ^{
 // <#code to be executed once#>
});
```

原理：判断一个全局静态变量的值，默认是 0，执行完`dispatch_once`后设置为 -1。dispatch_once() 执行时判断onceToken的值，是0才执行；用原子性操作block执行完成标记位，同时用信号量确保只有一个线程执行block，等block执行完再唤醒所有等待中的线程。

使用dispatch_once 实现单例：

```objc
// dispatch_once，线程安全，效率更高
+ (instancetype)sharedInstance {
    static id instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        if (instance == nil) {
            instance = [[self alloc]init];
        }
    });
    return instance;
}
```

使用dispatch_once实现单例可能导致死锁的原因： 在单例实例化返回单例对象之前，又调用了该单例对象。

要实现严格单例，应该禁用init方法，自定义私有的init方法；

## Dispatch After 延迟执行

#### dispatch_after：

延迟对应时间后，异步添加 block 到指定的 queue，注意：到该时间后只是将任务添加到队列，并不意味着一定开始执行；（比如是串行队列，可能存在排队），因此这个执行时间是不准确的。



##  Dispatch Barrier 栅栏：并发调度队列中执行的任务的同步点

在向并发调度队列添加栅栏时，该队列会延迟栅栏任务（以及栅栏之后提交的所有任务）的执行，直到所有先前提交的任务都执行完成为止。

### dispatch_barrier_sync：同步栅栏：

等待该函数传入的**队列中的任务都执行完毕**，再执行`dispatch_barrier_sync`函数中的任务以及后面的任务，会**阻塞**当前线程。

### dispatch_barrier_async：异步栅栏（重点）：

仍然是等前面的任务都执行完毕，再执行`dispatch_barrier_sync`函数中的任务以及后面的任务，但是是用的子线程，所以不会阻塞主线程（立即返回，继续执行下面的主线程代码）。

用途：

- **保证线程安全**：例如多个异步任务执行完之后再执行下一个操作；（上面的dispatch group也能实现同样的效果）；
- **实现读写安全**：例如实现safeArray, 让写操作放在barrier中，等待前面的“读”操作完成（而后续的“读”操作也会等到“写”操作完成）后才能继续执行。

注意：`dispatch_barrier_(a)sync`函数传入的的队列必须是自己**手动创建的并发队列**，如果传入的是全局并发队列或者串行队列，那么这个函数是没有栅栏的效果的，效果等同于`dispatch_(a)sync`函数。



## Dispatch Semaphore 信号量: 控制最大并发数量

**dispatch_semaphore_wait:  当信号量<= 0: 当前线程休眠等待；当信号量>=1 , 信号量-1 并开始执行任务；** 

**dispatch_semaphore_signal：信号量+1** （如果前一个值小于零，该函数将唤醒当前在dispatch_semaphore_wait中等待的线程）



## Dispatch Apply 多次执行

#### dispatch_apply(n, queue, block): 向队列中添加一个运行n次的block；

####  注意：

- **同步阻塞，直到运行完才返回**
- 使用**全局并发队列**时可以并发执行；
- 使用串行队列时，不会开启子线程，block 在主线程按串行执行；
- 使用并发队列时，不一定会开启子线程，block 不一定都在子线程执行，也可能都在主线程执行，取决于任务的耗时程度。



## Dispatch Source

用于监听系统底层对象(如文件描述符、Mach 端口、信号量等)，当这些对象有事件产生时会自动把事件的处理 block 函数提交到 dispatch 队列中执行；

用途：使用dispatch_source_create，设置回调，实现**准确的定时器**

```objc
dispatch_queue_t queue = dispatch_queue_create("myqueue", DISPATCH_QUEUE_SERIAL);
//创建定时器
dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
//设置时间（start:几s后开始执行； interval:时间间隔）
uint64_t start = 2.0; //2s后开始执行
uint64_t interval = 1.0; //每隔1s执行
dispatch_source_set_timer(timer, dispatch_time(DISPATCH_TIME_NOW, start * NSEC_PER_SEC), interval * NSEC_PER_SEC, 0);
//设置回调
dispatch_source_set_event_handler(timer, ^{
  NSLog(@"%@",[NSThread currentThread]);
});
//启动定时器
dispatch_resume(timer);
NSLog(@"%@",[NSThread currentThread]);

self.timer = timer;
```

