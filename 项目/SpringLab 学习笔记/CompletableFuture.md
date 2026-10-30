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

> [什么是 ExecutorService.submit()](<CompletableFuture/什么是 ExecutorService.submit().md>)

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

[executor.submit()内部 Lambda 详解](<CompletableFuture/executor.submit()内部 Lambda 详解.md>)

这里出现了与普通同步不同的地方：`submit()` 通常会很快返回一个 `Future`，而不会等待 Lambda 中的任务执行完成。

因此，主线程可以继续向下执行。
