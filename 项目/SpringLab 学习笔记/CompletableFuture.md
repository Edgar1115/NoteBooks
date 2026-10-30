# CompletableFuture

## Future -- FutureGetLab

> 运行示例见`FutureGetLab`，结合该类理解接口`Future`。

```Java

// FutureGetLab
// 先声明一个线程

ExecutorService executor = Executors.newSingleThreadExecutor(
    task -> new Thread(task, "ai-worker")
);
```

>[复习 Lambda 表达式](<./CompletableFuture/复习 Lambda 表达式.md>)
>
>[为什么这里是 task -\> new Thread(task, "ai-worker")](<./CompletableFuture/为什么这里是 task -%3E new Thread(task, "ai-worker").md>)

### 什么是 Future ？

`Future<T>` 是 Java 提供的一个接口，用来表示一个异步任务的执行结果。这个结果可能已经产生，也可能还没有产生。

假设要调用 AI，生成一个回答，这个过程需要 3 秒

普通的同步方法，主线程需要等待 `callAI()` 执行完成，才能继续执行下一行代码：

```Java

String result = callAI();
System.out.println(result);

```

如果使用线程池：

```Java

Future<String> future = executor.submit(() -> {
    Thread.sleep(3000);
    return "AI回答";
});

```

^b55cb0

> [什么是 ExecutorService.submit()](<./CompletableFuture/什么是 ExecutorService.submit().md>)

- `executor`：负责执行任务。
    
- `submit()`：把任务提交给线程池。
    
- `Future<String>`：代表这个异步任务未来的执行结果。

- `future`：保存返回的 `Future` 对象的引用。

>```Java
>Future<String> future
>```
>
>并不是说 `future` 已经包含字符串 `"AI回答"`。
>它表示的是：我们拿到了一个对象，可以通过它查询任务状态，或者在任务完成时获取结果。
>你可以把 `Future` 理解成一个异步任务的结果凭证。

```Java

// FutureGetLab

Future<String> future = executor.submit(() -> {
    log(startNanos, "模拟 AI 请求开始：准备耗时 3 秒");

    Thread.sleep(3000);

    log(startNanos, "模拟 AI 请求结束：返回结果");

    return "AI回答";
});
```

[executor.submit()内部 Lambda 详解](<./CompletableFuture/executor.submit()内部 Lambda 详解.md>)

这里出现了与普通同步不同的地方：`submit()` 通常会很快返回一个 `Future`，而不会等待 Lambda 中的任务执行完成。

因此，主线程可以继续向下执行。

继续看源码：

```Java
// FutureGetLab

log(
   startNanos,
   "submit() 已返回；future.isDone() = " + future.isDone()
);
```

`Future` 提供了一个方法：`boolean isDone();`用于判断异步任务是否已经结束。

- 返回 `false`：任务还没有结束。

- 返回 `true`：任务已经结束，包括正常完成、异常结束或被取消。`true`不代表一定成功。

Future.get() 是怎么工作的？

源码：

```Java
long beforeGetNanos = System.nanoTime();

String result = future.get();

long getWaitMillis =
    (System.nanoTime() - beforeGetNanos) / 1_000_000;
```

> future.get()阻塞的原理及影响，为什么 `getWaitMillis` 可以拿到等待时间-> [这里Callable方法应该怎么写](<./CompletableFuture/什么是 ExecutorService.submit()/这里Callable方法应该怎么写.md#^271855>)

---

### 单节Q&A

Q1: 所谓的任务未完成，`main`线程可以继续做自己的其他工作，等待`future`完成是什么意思？

A1: 首先，通过 `ExecutorService executor = Executros.newSingleThreadExecutor(...)` 创建一个由线程`ai-worker`，此时出现了两个线程，`main`线程和 `ai-worker`线程，在后续代码中，通过 `Future<String> future = executor.submit(...)`将任务提交给线程`ai-worker`，这个时候任务什么时候执行，将由[线程池和 JVM 的线程调度机制](<./CompletableFuture/线程池和 JVM 的线程调度机制.md>)决定，但是`main`线程此时已经可以不用管这个任务了，他可以去执行其他任务，等`ai-worker`这个线程去执行这个任务即可。

Q2: 但是 FutureGetLab 里面，写的是 String result = future.get()，那到这里，主线程就阻塞了嘛，那主线程不是一样会阻塞吗？他怎么去完成其他任务呢？

A2: 是的，在`main`线程里，执行到`future.get()`时，这里需要拿到任务的结果了，但是如果`ai-worker`没有完成任务，那么`main`线程一样要阻塞等待，”怎么去完成其他任务“，回答见 Q1/A1。

Q3: `ExecutorService executor = Executros.newSingleThreadExecutor(...)` 创建了线程 `ai-worker` 后，这个线程是马上开始工作吗？还是什么时候开始工作？

[FutureGetLab 涉及八股总结](<./CompletableFuture/FutureGetLab 涉及八股总结.md>)