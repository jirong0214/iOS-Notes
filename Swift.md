## Swift

### 结构化并发

回调： 非结构化；(回调时并发任务是否已执行完并不清楚)

1. 到底什么是结构化并发？

即使进行并发操作，也要保证控制流路径的单一入口和单一出口。程序可以产生多个控制流来实现并发，但是所有的**并发路径在出口时都应该处于完成** (或取消) 状态，并合并到一起。（这意味着我们得到返回值后，这个函数一定不是仍在运行的状态了）；

2. 异步函数是结构化并发的基础：为了将并发路径合并，程序需要具有**暂停等待其他部分的能力**。异步函数恰恰满足了这个条件：使用异步函数来获取暂停主控制流的能力，函数可以**执行其他的异步并发操作并等待它们完成**，最后主控制流和并发控制流统合后，从单一出口返回给调用者。

3. 结构化并发的用法：

   - **任务组（TaskGroup）**

     ```swift
     struct TaskGroupSample {
       func start() async {
         print("Start")
         // 1
         await withTaskGroup(of: Int.self) { group in
           for i in 0 ..< 3 {
             // 2
             group.addTask {
               await work(i)
             }
           }
           print("Task added")
     
           // 4  在这里显式地开始等待group中的任务完成....
           for await result in group {
             print("Get result: \(result)")
           }
           // 5
           print("Task ended")
         }
         print("End")
       }
     
       private func work(_ value: Int) async -> Int {
         // 3
         print("Start work \(value)")
         await Task.sleep(UInt64(value) * NSEC_PER_SEC)
         print("Work \(value) done")
         return value
       }
     }
     ```

     <img src="/Users/tianjirong/Library/Mobile Documents/com~apple~CloudDocs/笔记/assets/image-20221125172611482.png" alt="image-20221125172611482" style="zoom:33%;" />

     隐式等待：

     ```swift
     await withTaskGroup(of: Int.self) { group in
         for i in 0 ..< 3 {
             group.addTask {
                 await work(i)
             }
         }
         print("Task added")
     		// 即使group里没有主动调用await，也会在这里开始隐式等待
     }
     
     print("End")
     ```

   - **async let** (简化结构化并发的使用)

​	

```swift
func start() async {
  print("Start")
  async let v0 = work(0)
  async let v1 = work(1)
  async let v2 = work(2)
  print("Task added")
	//从这里开始等待
  let result = await v0 + v1 + v2
  print("Task ended")
  print("End. Result: \(result)")
}
```

结构化并发可以通过嵌套来更简单地控制并发任务；相比于使用信号量的方式，更简单；

