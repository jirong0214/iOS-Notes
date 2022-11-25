## Association

1. 用途：用于间接地给Category添加成员变量

2. 用法：给Category添加Property属性，并手动写get/set方法：

   key一般用@selector() 或者自定义字符串 或者直接用_cmd (当前selector)

```objective-c
@interface Person (Test)
@property (nonatomic, assign) int height;
@end

@implementation Person (Test)
- (void)setHeight:(int)height
{
    objc_setAssociatedObject(self, @selector(height), [NSNumber numberWithInt:height], OBJC_ASSOCIATION_ASSIGN);
}
- (int)height
{
    return [objc_getAssociatedObject(self, @selector(height)) intValue];
}
@end
```

3. **移除关联：**

   - 移除全部关联：**void** objc_removeAssociatedObjects(**id** object); 

     原理：在全局哈希表中移除(object, AssocaitionMap)的键值对

   - 移除单个key的关联：objc_setAssociatedObject(**self**, **@selector**(height), nil, OBJC_ASSOCIATION_ASSIGN);  

     原理：在全局哈希表中根据disguise(object)找到对应的哈希表,在此哈希表中移除(key, value)键值对；

4. **关联策略**

   | objc_AssociationPolicy            | 对应的属性修饰符  |
   | :-------------------------------- | ----------------- |
   | OBJC_ASSOCIATION_ASSIGN           | assign            |
   | OBJC_ASSOCIATION_RETAIN_NONATOMIC | strong, nonatomic |
   | OBJC_ASSOCIATION_COPY_NONATOMIC   | copy, nonatomic   |
   | OBJC_ASSOCIATION_RETAIN           | strong, atomic    |
   | OBJC_ASSOCIATION_COPY             | copy, atomic      |

5. **实现原理:**

- 关联对象并不是存储在关联对象本身内存中，而是存储在全局统一的一个容器中；（由 AssociationsManager 管理并在它维护的一个单例 Hash 表 AssociationsHashMap 中存储，使用了自旋锁保证线程安全。）

- disguise(object)作为key, 生成一个哈希表作为value （也就是说一个对象对应着一个哈希表）；

  其中这个哈希表里的key就是传入的key, value就是设置的value及policy的封装体;

  
