# 什么是 ExecutorService.submit()?

`ExecutorService` 中有一个重载方法：

```
<T> Future<T> submit(Callable<T> task);
```

可以这样理解：

| 部分                 | 含义              |
| ------------------ | --------------- |
| `Callable<T> task` | 传入一个可以返回结果的任务   |
| `submit(...)`      | 把任务交给线程池执行      |
| `Future<T>`        | 返回一个代表异步执行结果的对象 |
| `<T>`              | 任务返回值的类型        |

> [这里Callable方法应该怎么写](什么是%20ExecutorService.submit()?/这里Callable方法应该怎么写.md)

