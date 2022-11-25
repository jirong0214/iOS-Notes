## Category & Extension



### Category

1. 内容：实例方法、类方法、协议、属性；不能添加成员变量，但是可以通过关联对象间接实现；
2. 用途：

- 为类扩展功能(包括系统类)
- 将类按模块拆分，方便代码管理
- 使用Category创建对私有方法的声明

3. 特点：

   - 运行时决议：在运行时通过Runtime将分类合并到类中去；
   - 分类中的方法可以覆盖同名的宿主类方法；（因为消息传递过程中会优先找到分类中的方法并调用之，但是也可以手动通过Runtime找到类的方法列表，从中找到宿主类的方法实现进行调用。）

4. 加载过程:

   - Runtime加载宿主类的所有Category数据
   - 把所有Category数据(方法、属性、协议)合并到一个数组中
   - 将合并后的Category数据插入到宿主类数据的前面 (所以同名的会优先调用Category中的方法)

   

### Extension

1. 内容： **声明** **私有**属性、**私有**方法、**私有**成员变量 （直接写在宿主类的.m文件里）
2. 特点：
   - 编译时决议：在编译期间就将扩展的数据合并到宿主类中去
   - 可以声明私有的成员变量
   - 不能为系统类添加扩展 （没有.m源码）