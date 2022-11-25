## Runtime : C/C++/汇编编写的动态运行时库 

**用途：**

- 交换方法实现（用于hook）
- 利用关联对象给分类添加属性
- 遍历一个类的成员变量 （用于序列化反序列化、自动归档解挡等）
- 利用消息转发，解决unrecognized selector的问题

**主要API:**

- NSObject的方法
  - isKindOfClass、isMemberOfClass、respondsToSelector、conformsToProtocol、methodForSelector等...
- <objc/runtime.h>库的方法
  - 类相关：**objc_allocateClassPair**等.. 动态创建销毁类
  - 成员变量相关：**class_copyIvarList** 获取成员变量列表等
  - 属性相关：**class_addProperty**等.. 动态添加、替换属性
  - 方法相关：**class_replaceMethod**等.. 动态添加、替换方法
  - 关联对象相关：**objc_setAssociatedObject**...添加、获取、移除关联对象

### **数据结构**

OC对象：OC对象在runtime中就是objc_object结构体: `typedef struct objc_object *id;`

OC类：`typedef struct objc_class *Class; ` 它的父类也是objc_object

类信息的储存: 

- 一开始类的信息都存放在`class_ro_t`里，当程序运行时，经过一系列的函数调用栈，在`realizeClass()`函数中，将`class_ro_t`里的东西和分类的东西合并起来放到`class_rw_t`里，并让`bits`指向`class_rw_t`；类和扩展的方法都放在一个二维数组里

<img src="/Users/tianjirong/Library/Mobile Documents/com~apple~CloudDocs/笔记/assets/image-20221108224435577.png" alt="image-20221108224435577" style="zoom:33%;" />

- cache_t: 调用过的方法的使用哈希表缓存（局部性原理，空间换时间）

  哈希表缓存初始容量为4，超过3/4时进行缓存扩容；释放旧的缓存，使用新缓存；

### ISA：维护对象和类的关系，存储着类信息或元类信息

- 结构：

```objective-c
struct {
        uintptr_t nonpointer        : 1;  // 0：代表普通的指针，存储着 Class、Meta-Class 对象的内存地址
                                          // 1：代表优化过，使用位域存储更多的信息
        uintptr_t has_assoc         : 1;  // 是否有设置过关联对象，如果没有，释放时会更快
        uintptr_t has_cxx_dtor      : 1;  // 是否有C++的析构函数（.cxx_destruct），如果没有，释放时会更快
        uintptr_t shiftcls          : 33; // 存储着 Class、Meta-Class 对象的内存地址信息
        uintptr_t magic             : 6;  // 用于在调试时分辨对象是否未完成初始化
        uintptr_t weakly_referenced : 1;  // 是否有被弱引用指向过，如果没有，释放时会更快
        uintptr_t deallocating      : 1;  // 对象是否正在释放
        uintptr_t has_sidetable_rc  : 1;  // 如果为1，代表引用计数过大无法存储在 isa 中，那么超出的引用计数会存储在一个叫 SideTable 结构体的 RefCountMap（引用计数表）哈希表中
        uintptr_t extra_rc          : 19; // 里面存储的值是引用计数 retainCount - 1
    };
```

- 指针指向：

<img src="/Users/tianjirong/Library/Mobile Documents/com~apple~CloudDocs/笔记/assets/image-20221109141112129.png" alt="image-20221109141112129" style="zoom:33%;" />

instance.isa ->  class;  

class.isa -> meta-class;

meta-class.isa -> NSObject的meta-class

NSObject的meta-class.isa -> 指向自己

NSObject的meta-class.superclass -> NSObject

**class与meta-class**：将实例和类的相关方法列表以及构建信息区分开来，符合单一职责设计原则

- 底层都是objc_class (继承自objc_object)；
- 区别：class存储实例方法、属性、协议；meta-class存储类方法

- 调用一个类方法时的查找顺序: 通过`class`的`isa`指针找到`meta-class`，在`meta-class`中查找有无该类方法, 如果没有，再通过`meta-class`的`superclass`指针逐级查找父`meta-class`，一直找到基类的`meta-class`如果还没找到该类方法的话，就会去找**基类的`class`中同名的实例方法**的实现。

### Method (method_t struct)

```objc
struct method_t {
    SEL name;  // 方法名; 维护在一个全局的 Map 中，全局唯一，不同类中相同名字的方法的 SEL 是相同的
    const char *types;  // 编码（返回值类型、参数类型）
    IMP imp;   // 方法的实现的函数指针
};
```

### Type Encodings

Runtime把一个方法的返回值类型、参数类型通过字符串的形式描述；

https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html

在使用runtime进行消息转发时，会用到这个type encodings参数



### 消息机制  [消息发送 -> 动态方法解析 -> 消息转发]

##### **消息发送：**

**objc_msgSend(receiver, SEL, param1,params2...)**;  // 常规都是调用这个

objc_msgSend_stret;  // 向带有数据结构返回值的对象发送消息时；

objc_msgSendSuper; //  向super发送消息时；

objc_msgSendSuper_stret; // 向带有数据结构返回值的对象的super发送消息时；

<img src="/Users/tianjirong/Library/Mobile Documents/com~apple~CloudDocs/笔记/assets/image-20221109170033846.png" alt="image-20221109170033846" style="zoom:33%;" />

在确认receiver不是nil的情况下，通过isa指针找到本类的class (class / meta-class)；

1. 先查询**本类的cache**中是否存在该方法，如果没有就去**本类的class_rw_t**结构体的method_list列表中查询该方法；

2. 如果本类中没有，就递归地去**父类中分别查询缓存和方法列表**，若到最顶级的父类仍没找到，则进入动态解析机制；

注:  记得如果能在class_rw_t列表中找到的话要更新缓存; 列表如果是排序的就用二分查找，否则线性遍历；

##### **动态方法解析**（给1次机会向此类添加该方法）

<img src="/Users/tianjirong/Library/Mobile Documents/com~apple~CloudDocs/笔记/assets/image-20221109175544331.png" alt="image-20221109175544331" style="zoom:33%;" />

如何支持动态解析：根据方法类型（实例方法 or 类方法）重写 `+(BOOL)resolveInstanceMethod:(SEL)sel;` 或者`+(**BOOL**)resolveClassMethod:(SEL)sel;`方法，并在方法中使 **class_addMethod** 方法来动态添加方法；runtime就会去调用这个方法以完成动态添加方法，完成后重新回到上一步进行消息发送；

当动态解析1次后依然没能成功处理消息后，进入消息转发；

##### 消息转发

<img src="/Users/tianjirong/Library/Mobile Documents/com~apple~CloudDocs/笔记/assets/image-20221109184638048.png" alt="image-20221109184638048" style="zoom:33%;" />

- **Fast Forwarding**：将消息转发给另一个OC对象

  重写 `+/- (id) forwardingTargetForSelector:(SEL)sel`

- **Normal Forwarding**：启动一个完整消息转发

  重写 `+/- (NSMethodSignature *) methodSignatureForSelector:(SEL)aSelector`：

  （用处是定义一个方法签名(包括参数类型、返回值类型),因为只有SEL不足以定义一个方法）

  重写`+/- (void)forwardInvocation:(NSInvocation *)invocation`：

  这个invocation实例里封装了接收者、SEL、方法参数等信息，我们可以在这里做一些自定义逻辑：比如改变待处理消息的方法名、参数、并将其转发给其他对象；

- 如果没有重写 forwardInvocation 的方法，runtime就会调用doseNotRecognizeSelector方法，抛出unrecognized selector错误，彻底结束objc_msgSend流程；（Q：如何防止“调用无法识别的方法导致应用程序崩溃”？重写`doseNotRecognizeSelector`方法）

### self & super

1. OC 方法都带有两个隐式参数：`(id)self`和`(SEL)_cmd`；

- self: 是一个对象指针，指向当前消息接收者；当使用self调用方法时，runtime使用objc_msgSend方法进行消息发送流程
- super: 是一个编译器指令；当使用super调用方法时，runtime使用**objc_msgSendSuper**（汇编底层是objc_msgSendSuper2）方法进行消息发送流程；
- 使用super时，receiver消息接收者仍然是子类对象，只不过是直接从父类开始查找方法；

<img src="/Users/tianjirong/Library/Mobile Documents/com~apple~CloudDocs/笔记/assets/image-20221110120224692.png" alt="image-20221110120224692" style="zoom:33%;" />

如上图所示，使用self和super进行调用时，receiver都是self，而且最终找到的class/superclass方法都是在NSObject中找到的，故打印出来的是一样的结果；



### 相关问题：

#### 能否向编译后的类增加实例变量？能否向运行时动态创建的类增加实例变量？

不能，因为编译后的类的内存布局已确定；

能，但是要在注册类之前调用class_addIvar()完成成员变量的添加，一旦注册后也不能再添加

#### performSelector 方法的应用？

当该方法在编译时没有生成，而是在运行时动态生成的，就需要使用performSelector进行调用；

