## KVO 一种事件通知机制 <观察者模式>

### 作用：一个对象观察/监听另一个对象指定属性值的改变，当被观察对象属性值发生改变时，会触发`KVO`的监听方法来通知观察者;

- **注册**`KVO`监听；

  ```
  - (void)addObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath
   options:(NSKeyValueObservingOptions)options context:(nullable void *)context;
  ```

- **实现监听方法**以接收属性改变通知；

  ```
  - (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(**id**)object change:(NSDictionary *)change context:(void *)context
  ```

- **移除**`KVO`监听: 至少需要在观察者销毁之前移除监听，否则如果在观察者被释放后，再次触发`KVO`监听方法就会导致 **Crash**

  ```
  - (void)removeObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath;
  ```

**context**: 更精确的确定被观察对象属性;也可用来传值；苹果的推荐用法：用`context`来精确的确定被观察对象属性，使用唯一命名的静态变量的地址作为`context`的值。也可以为每个被观察对象属性设置不同的`context`，这样使用`context`就可以精确的确定被观察对象属性。

### 触发方式：

- #### 自动触发：

  - **点语法、setter、kvc的setValue:forKey/forFeyPath**；(直接修改下划线的成员变量不会触发KVO）
  - 如果是监听集合对象（Array/Set）的改变，需要通过`KVC`的`mutableArrayValueForKey:`等方法获得代理对象，并使用代理对象进行操作; （直接操作集合对象，不会触发KVO）

  - 可以在被观察对象的类中重写`+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key`方法来控制`KVO`的自动触发。 

- #### 手动触发：

  - 普通对象: 手动调用willChangeValueForKey / didChangeValueForKey; 
  - 集合对象: 手动调用 willChangeValuesAtIndexesforKey / didChangeValuesAtIndexesforKey 

  场景：直接修改下划线的成员变量时，通过手动调用以上方法可以实现手动触发KVO

- #### **如何实现新旧值相等时不触发KVO:**

  1. 先通过automaticallyNotifiesObserversForKey方法关闭该属性的自动KVO
  2. 手动重写setter方法，手动调用willChangeValueForKey方法仅在值有改变时触发KVO

- #### **如何手动触发集合对象的KVO：**

  需要实现以下方法：

  - 插入方法：`insertObject:in<Key>AtIndex`
  - 删除方法：`removeObjectFrom<Key>AtIndex:`
  - 替换方法：replaceObjectIn<Key>AtIndex:withObject:

### KVO使用注意事项：

- `KVO`的注册方法和移除方法应该是成对出现；忘记移除、重复移除都会导致Crash （分别常在init/dealloc中使用）
- 如何防止多次注册和移除相同KVO: 1. hook removeObserver方法，在里面加入@try catch, 防止抛出异常后直接crash；2.  利用 模型数组 进行存储记录；3. 利用 `observationInfo` 里私有属性。
- 当被观察对象属性发生改变时就会调用监听方法，如果观察者没有实现监听方法，就会Crash；
- 如果注册方法中`context`传的是一个对象，必须在移除观察之前**持有它的强引用**，否则在监听方法中访问`context`就可能导致`Crash`。
- 如果是监听集合对象（Array/Set）的改变，需要通过`KVC`的`mutableArrayValueForKey:`等方法获得代理对象，并使用代理对象进行操作; （直接操作集合对象，不会触发KVO）

### 原理：isa-swizzling

我们在为被观察对象`instance`添加`KVO`监听后，系统会在运行时利用`Runtime API`动态创建`instance`对象所属类`A`的子类`NSKVONotifying_A`，并且让`instance`对象的`isa`指向这个全新的子类，并重写原类`A`的被观察属性的`setter`方法来达到可以通知所有观察者对象的目的。

**重写的setter方法内容**:

- `willChangeValueForKey:`方法
-  父类原来的`setter`方法
- `didChangeValueForKey:`方法

除了重写了setter方法外，还重写了`class`、`dealloc`、`_isKVOA`这三个方法；

- class方法: 返回的是父类的`class`对象；因此调用class方法依然返回的是instance的类A；(目的是为了不让外界知道`KVO`动态生成类的存在；)

- `dealloc`方法：释放`KVO`使用过程中产生的东西；
- `_isKVOA`方法：用来标志它是一个`KVO`的类。