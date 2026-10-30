# FutureGetLab 涉及八股总结

> 按照代码从上至下阅读

---

```Java
ExecutorService executor =
    Executors.newSingleThreadExecutor(
        task -> new Thread(task, "ai-worker")
    );
```

线程池

八股1️⃣： 为什么要使用线程池？

```Plain
线程池主要有三个好处：

- 降低资源开销：复用已创建的工作线程，减少反复创建和销毁线程的成本。
    
- 控制并发数量：通过控制工作线程数量，避免创建过多线程。
    
- 统一管理任务：支持任务队列、拒绝策略、线程生命周期管理等。
```

对应本例：

`Executors.newSingleThreadExecutor()` 创建一个单线程线程池。

后续提交的任务由线程池安排执行，而不是每提交一个任务就创建一个新线程。

八股2️⃣：线程池和操作系统分别负责什么？

```Plain
- 线程池：管理工作线程，并决定任务由哪个工作线程处理或进入队列等待。
    
- JVM：提供 Java 线程抽象，并与操作系统协作。
    
- 操作系统调度器：负责安排可运行的平台线程获得 CPU 执行机会。
```

对应本例：

```Java
task -> new Thread(task, "ai-worker")
```

这里仅定义了线程创建方式。

线程池需要创建线程时，调用这个 `ThreadFactory`；创建出来的平台线程什么时候获得 CPU 执行机会，主要由操作系统调度。

八股3️⃣：线程池的七大核心参数是什么？

`ThreadPoolExecutor` 的七个构造参数：

|参数|作用|
|---|---|
|`corePoolSize`|核心线程数|
|`maximumPoolSize`|最大线程数|
|`keepAliveTime`|超出核心线程数的空闲线程存活时间（默认规则）|
|`unit`|存活时间单位|
|`workQueue`|任务等待队列|
|`threadFactory`|线程创建工厂|
|`handler`|拒绝策略|

对应本例：

`newSingleThreadExecutor()` 封装了线程池配置，核心线程数和最大线程数均为 1，采用无界任务队列。

传入的 Lambda 自定义了 `threadFactory`，把新工作线程命名为 `ai-worker`。

八股4️⃣：CPU 密集型和 I/O 密集型任务如何配置线程池？

```Plain
- CPU 密集型：主要消耗 CPU，线程数通常从 CPU 核数附近开始评估。
    
- I/O 密集型：存在大量网络、磁盘等等待，可以配置相对更多线程，但也必须考虑下游连接数、服务容量和资源限制。
```

对应本例：

模拟 AI 请求使用：

```Java
Thread.sleep(3000);
```

它用于模拟等待，不是实际 CPU 计算，也不是真实网络 I/O。真实 AI HTTP 请求通常属于 I/O 密集型任务，但不能仅凭本例推导出合理线程数。

```Java
Future<String> future = executor.submit(() -> {
```

八股5️⃣：execute() 和 submit() 有什么区别？

|`execute()`|`submit()`|
|---|---|
|返回 `void`|返回 `Future`|
|接收 `Runnable`|可接收 `Runnable` 或 `Callable`|
|不直接提供结果对象|可以通过 `Future` 获取结果|
|任务异常通常经工作线程的异常处理机制处理|任务异常通常保存在 Future 中，调用 `get()` 时暴露|

对应本例：

使用 `submit()` 是因为我们希望得到 AI 返回的 `"AI回答"`。

八股6️⃣：Runnable 和 Callable 有什么区别？

```Plain
- Runnable 的 run() 没有返回值，不能直接声明抛出受检异常。
    
- Callable<V> 的 call() 可以返回 V 类型的结果，并且可以声明抛出异常。
```

对应本例：

```Java
() -> {
    Thread.sleep(3000);
    return "AI回答";
}
```

由于 Lambda 返回 `String`，这里匹配的是 `Callable<String>` 的 `submit()` 重载。

八股7️⃣：线程池提交任务后的执行流程是什么？

```Plain
普通 ThreadPoolExecutor 在运行状态下，典型流程如下：

1. 当前工作线程数少于核心线程数，创建核心线程执行任务。
    
2. 否则尝试将任务加入工作队列。
    
3. 队列无法接收任务且线程数少于最大线程数，尝试创建非核心线程。
    
4. 仍无法处理，则执行拒绝策略。
```

对应本例：

```Java
executor.submit(...)
```

主线程将任务提交到线程池。

由于这是单线程线程池，同一时刻只能有一个任务在执行。如果工作线程正在执行其他任务，新任务会排队。

注意，`submit()` 是提交任务的入口，完整的线程池任务调度流程没有直接展示在这段代码里。

八股8️⃣：Future 和 FutureTask 是什么关系？

```Plain
Future 是接口，定义了获取异步结果、查询任务状态和取消任务等能力。

FutureTask 是 RunnableFuture 的实现类，而 RunnableFuture 同时继承 Runnable 和 Future。

因此，FutureTask 既可以作为任务执行，又可以保存任务结果和状态。

标准 ThreadPoolExecutor 通过 submit(Callable) 提交任务时，通常会包装成 FutureTask。
```

对应本例：

```Java
Future<String> future = executor.submit(...);
```

返回值的声明类型是 `Future<String>`，底层通常由 `FutureTask` 实现相关功能。

---

```Java
log(startNanos, "模拟 AI 请求开始：准备耗时 3 秒");

Thread.sleep(3000);

log(startNanos, "模拟 AI 请求结束：返回结果");

return "AI回答";
```

八股9️⃣：Java 线程有哪些状态？

Java 线程有六种状态：

| 状态              | 含义                       |
| --------------- | ------------------------ |
| `NEW`           | 线程创建但未启动                 |
| `RUNNABLE`      | 运行中或等待 CPU 执行机会          |
| `BLOCKED`       | 等待获得 `synchronized` 监视器锁 |
| `WAITING`       | 无期限等待特定事件                |
| `TIMED_WAITING` | 有时间限制的等待                 |
| `TERMINATED`    | 线程执行结束                   |

对应本例：

```Java
Thread.sleep(3000);
```

执行这句话的是 `ai-worker`。

它在休眠期间处于 `TIMED_WAITING` 状态。

休眠结束后，线程恢复可运行状态，继续执行后面的代码。

八股🔟：Thread.sleep() 和 Object.wait() 有什么区别？

```Plain
- Thread.sleep()：让当前线程休眠指定时间，不会释放已持有的监视器锁。
    
- Object.wait()：要求当前线程持有目标对象的监视器锁；调用后释放该对象的监视器锁，等待通知、中断或超时。
```

对应本例：

```Java
Thread.sleep(3000);
```

工作线程休眠 3 秒。

它不会因为主线程调用 `future.get()` 而停止执行；两个线程各自独立运行。

本例并未使用 `Object.wait()`，这里只是关联面试知识。

---

```Java
log(
    startNanos,
    "submit() 已返回；future.isDone() = " + future.isDone()
);
```

八股 ⑪：Future.isDone() 有什么作用？

```Plain
isDone() 用来检查异步任务是否已经结束。

- false：任务尚未结束。
    
- true：任务已经结束，可能是正常完成、异常结束或被取消。

isDone() 本身不会阻塞调用线程。
```

对应本例：

主线程提交任务后，立即检查 `future.isDone()`。

由于工作线程需要休眠 3 秒，这时通常返回 `false`。

但 `isDone()` 的结果取决于实际执行时机，并不保证第一次一定是 `false`。

---

```Java
log(startNanos, "主线程继续处理其他工作（约 1 秒）");

Thread.sleep(1000);

log(startNanos, "主线程其他工作完成；future.isDone() = " + future.isDone());
```

八股⑫：并发和并行有什么区别？

```Plain
- 并发：多个任务在同一时间段内交替或重叠推进。
    
- 并行：多个任务在同一时刻真正运行，通常依赖多个 CPU 核心。
```

对应本例：

`main` 和 `ai-worker` 可以同时推进各自的任务。

`ai-worker` 正在执行耗时 3 秒的模拟 AI 请求，而 `main` 不必马上等待结果。

不过这里的 `Thread.sleep(1000)` 只是模拟其他业务花费了 1 秒，主线程实际上是在休眠，不是在执行其他业务计算。

这段代码主要用来演示：提交异步任务后，主线程仍可以继续执行后面的代码。

---

```Java
log(startNanos, "即将调用 future.get()");

long beforeGetNanos = System.nanoTime();

String result = future.get();

long getWaitMillis =
    (System.nanoTime() - beforeGetNanos) / 1_000_000;

log(startNanos, "get() 返回，等待约 " + getWaitMillis + " ms；结果：" + result);
```

八股⑭：Future.get() 为什么会阻塞？

```Plain
Future.get() 用于获取异步任务结果。

- 如果任务已经正常完成，直接返回结果。
    
- 如果任务尚未完成，调用 get() 的线程就会等待结果。
    
- 如果任务执行异常，get() 会抛出 ExecutionException。
    
- 如果任务被取消，get() 会抛出 CancellationException。
```

对应本例：

调用 `get()` 的是 `main`，因此任务尚未完成时，被阻塞的是主线程。

`ai-worker` 仍然能够继续执行任务。

这里需要注意一个面试细节：

主线程在 `get()` 处“阻塞”，不等于它的 Java 线程状态是 `BLOCKED`。

在此例常见的 `FutureTask` 实现中，`main` 等待未完成结果时通常是 `WAITING` 状态。

八股⑭：FutureTask 如何实现线程等待和唤醒？

```Plain
FutureTask 内部维护任务状态、计算结果或异常，以及等待任务完成的线程。

当调用 get() 时：

1. 首先检查任务是否已经完成。
    
2. 如果没有完成，将调用线程加入等待机制。
    
3. 等任务完成后，保存结果或异常。
    
4. 唤醒等待结果的线程。
    
5. get() 根据任务最终状态返回结果或抛出异常。
    
现代 JDK 的 FutureTask 实现会使用 CAS、等待节点以及 LockSupport.park()/unpark() 等机制进行线程协调。
```

对应本例：

```Java
String result = future.get();
```

可以把它理解为：

`main` 等待 → `ai-worker` 完成计算 → FutureTask 记录结果并唤醒等待者 → `main` 获取 `"AI回答"`。

---

```Java
finally {
    executor.shutdown();
}
```

八股⑮：shutdown() 和 shutdownNow() 有什么区别？

```Plain
- shutdown()：平滑关闭，不再接收新任务，已经提交的任务继续执行。
    
- shutdownNow()：尝试中断正在执行的任务，并返回尚未开始执行的任务列表；不能保证任务一定停止。
```

对应本例：

```Java
executor.shutdown();
```

线程池不再接收新的任务，但不会立即强制终止已经提交的任务。

延伸八股：

```Plain
线程池有哪些拒绝策略？

四种常见策略：

- AbortPolicy：抛出异常。
    
- CallerRunsPolicy：由提交任务的线程执行（线程池未关闭时）。
    
- DiscardPolicy：丢弃任务。
    
- DiscardOldestPolicy：丢弃队列头部任务，再尝试提交。
```

本例没有实际触发拒绝策略，因此这一题属于延伸知识点。