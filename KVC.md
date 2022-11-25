## KVC : 通过字符串间接获取属性

### 主要API: 

- valueForKey
- valueForKeyPath
- setValueForKey
- setValueForKeyPath

key：属性的字符串：例如`[myAccount setValue:@(100.0) forKey:@"currentBalance"];`

keyPath：用.分隔的属性字符串：例如 `[myAccount setValue:@"地址" forKeyPath:@"owner.address.street"];`

### 聚合运算符：

用于读取集合中的对象的某个属性并**返回一个值**；例如计算@avg/@count/@sum/@max/@min等

语法： @运算符.属性名；

```objc
// 计算 transactions 数组中 amount 的平均值。
NSNumber *transactionAverage = [self.transactions valueForKeyPath:@"@avg.amount"];
// transactionAverage 格式化的结果为 $ 456.54。
```

### 数组运算符：

用于读取集合中的对象的某个属性并**返回一个数字**；例如计算@unionOfObjects/@distinctUnionOfObjects等

```objc
// 获取集合中的所有 payee 对象。
NSArray *payees = [self.transactions valueForKeyPath:@"@unionOfObjects.payee"];
// payees 数组包含以下字符串：Green Power, Green Power, Green Power, Car Loan, Car Loan, Car Loan, General Cable, General Cable, General Cable, Mortgage, Mortgage, Mortgage, Animal Hospital。
```

### **嵌套运算符：**

用于读取集合中的每个集合中的每个元素的右键路径指定的属性，并根据运算符返回一个`NSArray`或`NSSet`实例；@unionOfArrays/@distinctUnionOfArrays/@distinctUnionOfSets..

```objc
// 获取 arrayOfArrays 集合中的每个集合中的所有 payee 对象。
NSArray *collectedPayees = [arrayOfArrays valueForKeyPath:@"@unionOfArrays.payee"];
// collectedPayees 数组包含以下字符串：Green Power, Green Power, Green Power, Car Loan, Car Loan, Car Loan, General Cable, General Cable, General Cable, Mortgage.....
```

**注**：如果集合中的对象都是`NSNumber`，右键路径可以用`self`。

### 自定义运算符：

自己在集合的分类中添加_medianForKeyPath, 即可添加一个自定义的中位数运算符；

```objc
@interface NSArray (HTOperator)
- (NSNumber *)_medianForKeyPath:(NSString *)keyPath;
@end

#import "NSArray+HTOperator.h"
@implementation NSArray (HTOperator)
- (NSNumber *)_medianForKeyPath:(NSString *)keyPath {
   // 具体逻辑
}
```

```objc
NSArray *array = @[@9, @7, @8, @2, @6, @3];
NSNumber *num = [array valueForKeyPath:@"@median.self"];
```

### 使用valueForKey/setValueForKey时的自动装箱拆箱

当进行取值如`valueForKey:`时，如果返回值非对象，会使用该值初始化一个`NSNumber`对象(自动装箱)

当进行赋值如`setValue:forKey:`时，如果`key`的数据类型非对象，则会发送一条`<type>Value`消息给`value`对象以提取基础数据，然后赋值给`key`（自动拆箱）

注意：因为`Swift`中的所有属性都是对象，所以这里仅适用于`Objective-C`属性。`setValue:forKey:`时，如果`key`的数据类型是非对象类型，则`value`就禁止传`nil`。否则会Crash。（自动拆箱失败...）

### 属性验证

查看消息接收者类中是否实现了遵循命名规则为`validate<Key>:error:`的方法，如果有的话就返回调用该方法的结果；默认不验证；

```objc
- (BOOL)validateValue:(id  _Nullable *)value 
               forKey:(NSString *)key 
                error:(NSError * _Nullable *)error;
```



## 原理

### 搜索模式：

- Setter：

  - ① 按照`set<Key>:`、`_set<Key>:`顺序查找方法。
    如果找到就调用并将`value`传进去（根据需要进行数据类型转换），否则执行②。
  - ② 查看消息接收者类的`+accessInstanceVariablesDirectly`方法的返回值（默认返回`YES`）。如果返回`YES`，就按照`_<key>`、`_is<Key>`、`<key>`、`is<Key>`顺序查找成员变量（同 基本的 Getter 搜索模式）。如果找到就将`value`赋值给它（根据需要进行数据类型转换），否则执行③。如果`+accessInstanceVariablesDirectly`方法返回`NO`也执行③。
  - ③ 调用`setValue:forUndefinedKey:`方法，该方法抛出异常`NSUnknownKeyException`，并导致程序`Crash`。这是默认实现，我们可以重写该方法根据特定`key`做一些特殊处理。

  数据类型转换 : 如果取到的值是一个对象指针，即获取的是对象，则直接将对象返回。

  - 如果取到的值是一个`NSNumber`支持的数据类型，则将其存储在`NSNumber`实例并返回。
  - 如果取到的值不是一个`NSNumber`支持的数据类型，则转换为`NSValue`对象, 然后返回。

- Getter: 在Setter模式的基础上多了数组相关的搜索；（比较复杂）

- NSMutableArray：（比较复杂）

  返回一个代理对象，来响应发送给`NSMutableArray`的消息

### 异常处理

① 根据`KVC`搜索规则，**当没有搜索到对应的`key`或者`keyPath`相关方法或者变量时**，会调用对应的异常方法`valueForUndefinedKey:`或`setValue:forUndefinedKey:`，这两个方法的默认实现是抛出异常`NSUnknownKeyException`，并导致程序Crash

② 当进行赋值如`setValue:forKey:`时，**如果`key`的数据类型是非对象类型，则`value`就禁止传`nil`**。否则会调用`setNilValueForKey:`方法，该方法的默认实现是抛出异常`NSInvalidArgumentException`，并导致程序`Crash`。

我们可以通过重写以上方法来手动处理异常，而不是直接让程序Crash；



#### 通过 KVC 修改属性会触发 KVO 吗？会

#### KVC 违背面向对象的编程思想吗？是

`valueForKey:`和`setValue:forKey:`这里面的`key`是没有任何限制的，当我们知道一个类或实例它内部的私有变量名称的情况下，我们在外界可以通过已知的`key`来对它的私有变量进行访问或者赋值的操作。从这里来看是违背OOP的。



## 用途

1. ##### 访问和修改私有变量 （例如，修改一些控件的内部属性， 可以先通过runtime的class_copyIvarList拿到所有成员变量，再通过kvc进行访问和修改值）

2. ##### Model和字典转换 （用dictionaryValueForKey方法..）