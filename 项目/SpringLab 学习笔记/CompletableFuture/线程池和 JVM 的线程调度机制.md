# 线程池和 JVM 的线程调度机制涉及的八股

Java 并发八股 ：

1. 线程基础：线程是什么、谁负责调度、为什么会阻塞。
2. 线程池机制：线程池有什么用、如何提交任务、如何安排工作线程执行。
3. 异步任务与 Future：`Runnable`、`Callable`、`Future` 的区别，以及 `get()` 的阻塞机制。

 ---

## 线程基础

### 1. 进程和线程有什么区别？

进程是程序的一次运行实例，是操作系统进行资源分配与隔离的重要单位；线程是进程内部的执行单位，也是 CPU 调度的重要单位。

同一进程中的线程共享堆内存等资源，但各自拥有独立的栈和程序计数器。

结合 `FutureGetLab`：

```Plain
SpringLab（一个 JVM 进程）
    │
    ├── main       主线程
    │
    └── ai-worker  工作线程
```

`main` 和 `ai-worker` 属于同一个 JVM 进程，可以共享对象，但各自拥有独立的执行路径。

### 2. 并发和并行有什么区别？

并发（Concurrency）指多个任务在同一时间段内推进；并行（Parallelism）指多个任务在同一时刻真正执行。

单核 CPU 可以通过线程切换实现并发，多核 CPU 则可以让不同线程在不同核心上并行执行。

`FutureGetLab` 中，`main` 和 `ai-worker` 可以并发推进，但是否在某一时刻真正并行，还要看 CPU 资源和调度情况。

### 3. Java 线程有哪些状态？

Java `Thread.State` 有六种状态：

| 状态              | 含义                       | 例子                      |
| --------------- | ------------------------ | ----------------------- |
| `NEW`           | 创建但尚未启动                  | `new Thread(...)`       |
| `RUNNABLE`      | 正在运行或等待 CPU              | 正常执行代码                  |
| `BLOCKED`       | 等待获取 `synchronized` 监视器锁 | 锁被其他线程持有                |
| `WAITING`       | 无期限等待特定事件                | `Future.get()` 等待未完成的任务 |
| `TIMED_WAITING` | 有期限的等待                   | `Thread.sleep(3000)`    |
| `TERMINATED`    | 线程执行结束                   | `run()` 执行完毕            |

注意：这是 Java 的线程状态，而不是操作系统的完整调度状态。

对于你当前的代码，最需要注意：

```Java
Thread.sleep(3000);
```

`ai-worker` 进入 `TIMED_WAITING`。

```Java
String result = future.get();
```

如果任务尚未完成，在当前常见的 `FutureTask` 实现中，`main` 通常进入 `WAITING`。

这里有一道经典陷阱：日常口语说 `get()` 会阻塞主线程，不代表 `Thread.State` 一定是 `BLOCKED`。 Java 的 `BLOCKED` 特指等待获取监视器锁。

### 4. sleep()、wait()、get() 有什么区别？

| 方法               | 主要作用                   | 是否释放已有的监视器锁    |
| ---------------- | ---------------------- | -------------- |
| `Thread.sleep()` | 让当前线程休眠一段时间            | 不释放            |
| `Object.wait()`  | 等待其他线程通知或超时，需要先持有对象监视器 | 释放调用对象的监视器锁    |
| `Future.get()`   | 等待异步任务结果               | 不会自动释放调用线程持有的锁 |

这部分掌握到这里即可。后续学习 `synchronized` 时，再详细理解 `wait()` 的锁机制。

---

## 线程池

### 5. 为什么要使用线程池？

线程池主要有三个作用：

- 线程复用：避免为每个任务重复创建和销毁线程。
    
- 控制并发：限制同时执行任务的工作线程数量。
    
- 统一管理：支持任务排队、拒绝策略、生命周期管理和监控。

之前的：

```Java
Executors.newSingleThreadExecutor()
```

它为什么叫单线程线程池？

因为同一时刻最多只有一个工作线程执行任务。

假设提交三个任务：

```Java
executor.submit(taskA);
executor.submit(taskB);
executor.submit(taskC);
```

如果 `taskA` 还没有执行完，`taskB` 和 `taskC` 就会排队等待，而不是同时再创建两个工作线程。

### 6. 线程池的七大核心参数是什么？

面试的高频背诵题。`ThreadPoolExecutor` 的七个构造参数如下：

| 参数                | 作用                         |
| ----------------- | -------------------------- |
| `corePoolSize`    | 核心线程数                      |
| `maximumPoolSize` | 最大线程数                      |
| `keepAliveTime`   | 超出核心线程数的空闲工作线程可保留的时间（默认规则） |
| `unit`            | 空闲存活时间单位                   |
| `workQueue`       | 等待执行任务的队列                  |
| `threadFactory`   | 创建线程的工厂                    |
| `handler`         | 任务拒绝策略                     |

例如：

```Java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    2,                              // 核心线程数
    4,                              // 最大线程数
    60,                             // 空闲存活时间
    TimeUnit.SECONDS,               // 时间单位
    new ArrayBlockingQueue<>(10),   // 任务队列
    Executors.defaultThreadFactory(), // 线程工厂
    new ThreadPoolExecutor.AbortPolicy() // 拒绝策略
);
```

其中 `threadFactory`：

```Java
task -> new Thread(task, "ai-worker")
```

下一步主要需要理解 `workQueue` 和 `maximumPoolSize` 的作用。

### 7. 一个任务提交到线程池后，怎么执行？

![](<./附件/Pasted image 20261030214918.png>)

面试口诀：先核心线程，再任务队列，再最大线程，最后拒绝。

这是普通 `ThreadPoolExecutor` 的典型流程，不意味着每个新任务都一定先排队。

对于 `FutureGetLab` 使用的 `newSingleThreadExecutor()`，由于只有一个工作线程，其他任务会进入一个无界队列等待执行。

### 8. 线程池的四种拒绝策略是什么？

| 策略                    | 行为                                   |
| --------------------- | ------------------------------------ |
| `AbortPolicy`         | 默认策略，抛出 `RejectedExecutionException` |
| `CallerRunsPolicy`    | 线程池未关闭时，由提交任务的线程自己执行                 |
| `DiscardPolicy`       | 直接丢弃任务                               |
| `DiscardOldestPolicy` | 丢弃队列头部的任务，再尝试提交                      |

其中 `CallerRunsPolicy` 很值得理解。

假设 `main` 提交任务时，线程池已经满了，而且使用 `CallerRunsPolicy`：

```Java
executor.execute(task);
```

那么该任务可能直接由 `main` 执行。

这就意味着 提交任务的主线程也可能被耗时任务占用，不再像正常异步提交时那样很快返回。

## Future

### 9. Runnable 和 Callable 有什么区别？

|特性|`Runnable`|`Callable<V>`|
|---|---|---|
|抽象方法|`run()`|`call()`|
|是否有返回值|没有|有|
|是否可以声明抛出受检异常|`run()` 不能|`call()` 可以|
|常见用途|无结果任务|需要返回结果的任务|

在 `FutureGetLab` 里写的：

```Java
Future<String> future = executor.submit(() -> {
    Thread.sleep(3000);
    return "AI回答";
});
```

因为这个 Lambda 返回了 `String`，所以匹配的是 `Callable<String>` 类型的 `submit()` 重载。

### 10. execute() 和 submit() 有什么区别？

`execute()` 和 `submit()` 都可以向线程池提交任务，但有两个主要区别：

| 对比   | `execute()`      | `submit()`                                    |
| ---- | ---------------- | --------------------------------------------- |
| 返回值  | `void`           | `Future`                                      |
| 接收任务 | `Runnable`       | `Runnable`、`Callable`                         |
| 获取结果 | 不直接提供 Future     | 可以通过 Future                                   |
| 任务异常 | 通常由工作线程的异常处理机制处理 | 通常保存于 Future，`get()` 时抛出 `ExecutionException` |

实际源码中，常见 `submit()` 实现会把任务包装成 `FutureTask`，再交给执行器执行。

### 11. Future 为什么能获取异步执行结果？

`Future` 是一个表示异步计算结果的接口。线程池执行任务后，会由对应的 Future 实现对象保存任务的完成状态、结果或异常。

调用方可以通过 `Future` 查询任务是否完成，也可以在需要结果的时候调用 `get()`。

在常见的 `FutureTask` 实现中，`get()` 会检查任务状态。如果任务未完成，就让调用线程等待；任务完成时，等待的线程可以被唤醒并获取结果。

因此，任务执行与结果获取被分离了。

### 12. Future.get() 为什么会阻塞？

```Java
Future<String> future = executor.submit(() -> {
    Thread.sleep(3000);
    return "AI回答";
});

String result = future.get();
```

`submit()` 提交任务后，`main` 不必等待任务结束。

但 `future.get()` 要求获取结果，所以如果任务尚未完成，调用它的 `main` 就必须等待。

这里还要掌握几种异常：

| 情况              | `get()` 的结果                |
| --------------- | -------------------------- |
| 正常执行完毕          | 返回结果                       |
| 任务执行失败          | 抛出 `ExecutionException`    |
| 等待的线程被中断        | 抛出 `InterruptedException`  |
| 任务被取消           | 抛出 `CancellationException` |
| 限时 `get()` 等待超时 | 抛出 `TimeoutException`      |

如果调用 `future.cancel(true)`，对于常见的线程池任务，会尝试通过中断来取消正在执行的任务，但不能保证底层业务立即停止。