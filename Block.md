## Block: 封装了函数调用以及调用环境的 OC 对象

<img src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/65ba901acf26484194b8adce70382114~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image" alt="img" style="zoom: 33%;" />

1. **声明时用copy**

```
@property (nonatomic, copy) void(^myBlock1)(void);

typedef void(^BlockType)(void); // BlockType:类型别名
@property (nonatomic, copy) BlockType myBlock2;
```

2. **数据结构**

<img src="/Users/tianjirong/Library/Mobile Documents/com~apple~CloudDocs/笔记/assets/image-20221107231030282.png" alt="image-20221107231030282" style="zoom:22%;" />

定义Block: 调用构造函数，传入参数（代码、长度），返回返回值的地址；

调用Block: 使用*__block_impl* 中的FuncPtr函数地址直接调用；

3. **变量捕获**

<img src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/797e6dd11f2c4a4d8a530aaf04ffe680~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image" alt="block 变量捕获机制.png" style="zoom:33%;" />

注: auto就是普通的局部变量，可能会被释放，所以使用值传递进行捕获；

静态变量一直处于内存中，所以使用引用传递；

全局变量不需要捕获(因为任何作用域都可以访问)；

| 函数                                              | 调用时机                |
| ------------------------------------------------- | ----------------------- |
| copy 函数：根据strong/week等对该变量进行强/弱引用 | 栈上的 block 复制到堆时 |
| dispose 函数：对该变量进行release                 | 堆上的 block 被废弃时   |

4. **__block**

block内部若想要修改捕获的外部变量(赋值)：1. 用全局变量 2.用静态变量 3. **用__block修饰**

__block做了什么: 编译器将该变量包装成了一个(Block_byref_age_0)引用类型的对象

__block变量p的内存管理: block在栈上时不会强引用p，当block被复制到堆上时(被调用时)会强引用p, 当block从堆中移除时会release p;

__block变量里的 ` __forwarding` 指针指向自己，以确保访问的变量都是在堆上的那个变量；

__block修饰的值类型变量(int)与普通的对象类型变量的关系与区别:

- 相同点: 在栈上时都不会对他们进行强引用；从堆中移除时都会使用despose() 对其进行release；
- 不同点: 对象类型可以根据变量的修饰符（`__strong`、`__weak`、`__unsafe_unretained`）做出相应的操作，形成强引用或弱引用；__block变量则只能形成强引用；

- 注：被__block修饰的对象类型的表现与普通的对象类型变量是一样的(都是引用传递)；

在ARC下，`strong`或者`copy`都会对 block 进行强引用，都会自动将 block 从栈 copy 到堆上，但是一般还是用copy;

5. Block的**循环引用**问题：

   - 当当前对象持有block,而block又强引用了当前对象时，便会循环引用。

   - 解决办法: 

     1. 用`__weak`或者`__unsafe_unretained`修饰self：

     ` __weak typeof(self) weakSelf = self;`

     2. 用__block 修饰self, 并在block里面用完之后置为nil，但是这种方法如果一直没有调用block，则还是会循环引用；

6. **weak - strong** ：解决通过弱指针访问对象成员时可能会提前释放的问题：

   ```
   __weak typeof(self) weakSelf = self;
       self.block = ^{
           __strong typeof(weakSelf) strongSelf = weakSelf; //避免执行过程中提前释放
           NSLog(@"%d",strongSelf->age);
       };
   ```

   

​     