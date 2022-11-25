## load & initialize

### + (void) load 方法

1. 调用时机：Runtime加载所有重写了load方法的类、Category时进行调用；（就算没用到这些类，也会加载这些类，并运行load方法）
2. 调用方式：
   - 系统调用：通过函数地址调用
   - 开发者手动调用：通过objc_msgSend调用
3. 调用顺序：先按照编译顺序，再按照父类(loadable_classes) -> 子类 (loadable_classes) -> 分类(loadable_categories) ； 

### \+ (void) initialize 方法

1. 调用时机：在该类首次接收到消息时调用（所以要用到这些类，才会调用）；

2. 调用方式：消息传递 objc_MsgSend; （这意味着分类的会覆盖宿主类的）

3. 调用顺序: 总是先调用父类initialize，再调用子类initialize；

   当子类没有实现自己的initialize方法时，会去调用父类的initialize方法；

   所以父类的initialize方法可能会被调用多次；(但是父类只会初始化1次，后面的调用是为了子类的初始化)

## 

### 对比

| 区别     | load                                                         | initialize                                                   |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 调用时刻 | 在`Runtime`加载类、分类时调用 （不管有没有用到这些类，在程序运行起来的时候都会加载进内存，并调用`+load`方法）。  每个类、分类的`+load`，在程序运行过程中只调用一次（除非开发者手动调用）。 | 在`类`第一次接收到消息时调用。  如果子类没有实现`+initialize`方法，会调用父类的`+initialize`，所以父类的`+initialize`方法可能会被调用多次，但不代表父类初始化多次，每个类只会初始化一次。 |
| 调用方式 | ① 系统自动调用`+load `方式为直接通过函数地址调用； ② 开发者手动调用`+load `方式为消息机制`objc_msgSend`函数调用。 | 消息机制`objc_msgSend`函数调用。                             |
| 调用顺序 | ① 先调用类的`+load `，按照编译先后顺序调用（先编译，先调用），调用子类的`+load `之前会先调用父类的`+load `； ② 再调用分类的`+load `，按照编译先后顺序调用（先编译，先调用）（注意：分类的其它方法是：后编译，优先调用）。 | ① 先调用父类的`+initialize`， ② 再调用子类的`+initialize` （先初识化父类，再初始化子类）。 |
