# CompletableFuture

## Future -- FutureGetLab

> 运行示例见`FutureGetLab`，结合该类理解接口`Future`。

```Java
// FutureGetLab
// 先声明一个线程

ExecutorService executor = Executors.newSingleThreadExecutor(
	// 代码 A
    task -> new Thread(task, "ai-worker")
);
```

>[复习 Lambda 表达式](<./CompletableFuture/复习 Lambda 表达式.md>)
>
>[代码 A 详解](CompletableFuture/为什么这里是%20task%20->%20new%20Thread(task,%20"ai-worker").md)

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

A3: 

Q4：我已经理解异步这一点了，但是如果没有 future，就不能执行 executor.submit(...)了吗？

A4：可以，即使不用`Future`变量，也可以执行`executor.submit(...)`，但是`Future`负责表示任务的执行结果。所以如果不用 `Future` 接收，一样异步执行。

Q4(追问 1)：这里是因为我想拿到一个返回值，但是要异步进行，所以才用一个 Future？那如果我直接用一个 String 对象接收呢？

A4(追问 1)：对，因为希望任务异步执行，同时又希望在将来获取任务的返回值，所以使用了 `Future<String>`。不能用 `String` 接收，因为 `executor.submit(...)` 的返回值是 `Future<String>` 不是 `String`。

Q4(追问 2)：所以如果我不用 future 接收，是不是我就拿不到 executor.submit(...)的返回值了？

A4(追问 2)：这里确实拿不到 executor.sub(...)的返回值，因为它返回的是一个`Future<String>`对象，但是也可以用一个`String`对象接收`Lambda`的返回值，格式如下：

```Java
String result = executor.submit(() -> {
    return "AI回答";
}).get();
```

但是这样，主线程`main`就会立刻调用`.get()`，如果任务没有完成，就会阻塞，直到任务完成。

[FutureGetLab 涉及八股总结](<./CompletableFuture/FutureGetLab 涉及八股总结.md>)

### 传统`Future`的局限

#### 一、不能方便地手动设置任务结果。

传统的`Future`：

```Java
ExecutorService executor =
    Executors.newSingleThreadExecutor();

Future<Integer> future = executor.submit(() -> {
    return 100;
});

Integer result = future.get();
```

这里的流程是：

```Plain
executor.submit()
       │
       ▼
线程池执行 Callable
       │
       ▼
Callable 返回 100
       │
       ▼
Future 保存结果
       │
       ▼
future.get() 获得 100
```

使用传统 `Future` 时，通常是我们负责定义任务，线程池负责执行任务，并在任务完成后保存返回值。我们不需要，也不能通过 `Future` 接口直接设置这个返回值。

上面的示例代码，涉及三个角色：

```Plain
1. 我们（程序员）：定义任务，告诉线程池要执行什么。
    
2. 线程池（ExecutorService）：安排线程执行任务，并管理任务的完成状态和结果。
    
3. Future：代表这个任务未来的执行结果，供我们查询和获取。
```

将上述的流程更具体一点：

```Plain
程序员
   │
   │ 定义 Callable 任务
   ▼
executor.submit(...)
   │
   ├── 返回 Future 对象给调用者
   │
   └── 线程池安排工作线程
             │
             ▼
        执行 Callable
             │
             ▼
          return 100
             │
             ▼
        任务执行框架记录结果
             │
             ▼
        Future 变成已完成状态
             │
             ▼
        future.get()
             │
             ▼
          得到 100
```

这里最终`Future` 得到了 `100`，这是`Callable`任务的返回值，我们并没有手动写 `future.complete(100);`，因为传统的 `Future` 接口根本没有 `complete()` 方法。

**`CompletableFuture`有什么不同？**

示例：

```Java
CompletableFuture<Integer> future =
    new CompletableFuture<>();

future.complete(100);

Integer result = future.get();

System.out.println(result); // 100
```

这里的区别在于，没有`executor.submit(...)`，也没有`Callable<Integer>`，而是直接`future.complete(100);`，这里就叫**手动完成结果**。

`CompletableFuture` 让我们不必依赖 `ExecutorService.submit()` 来产生 Future 的结果。

> 传统 `Future` 接口没有公开的 `complete()` 方法，但可以通过 `FutureTask` 等实现类间接控制任务完成。这里说的是它不提供像 `CompletableFuture` 这样直接、方便的手动完成 API。

#### 二、不能直接定义任务完成后要做什么，以及多个任务之间如何配合

假设现在有两个任务：

任务 A： 调用 AI 获取评分。

任务 B： 将评分乘以 2。

而且，B 必须等 A 完成之后才能执行。

传统`Future`：

```Java
Future<Integer> future = executor.submit(() -> {
    return 80;
});

// 等待任务 A 完成
Integer score = future.get();

// 执行任务 B
Integer result = score * 2;

System.out.println(result); // 160
```

执行流程：

```Plain
任务 A：计算评分
       │
       ▼
future.get()
       │
       ▼
得到 80
       │
       ▼
任务 B：80 × 2
       │
       ▼
得到 160
```

如果 A 没完成，那么当前线程就会阻塞等待。

我们需要自己编写后续处理逻辑：

```Java
Integer score = future.get();
Integer result = score * 2;
```

用`CompletableFuture`：

```Java
CompletableFuture<Integer> future =
    CompletableFuture.supplyAsync(() -> {
        return 80;
    });

CompletableFuture<Integer> resultFuture =
    future.thenApply(score -> score * 2);
```

这里涉及到两个新方法

① `CompetableFuture.supplyAsync(...)`：类似于`ExecutorService.submit(...)`，都会将任务交给线程池执行，并立即返回一个代表未来结果的对象，而不是等待任务执行完成。

| 对比     | `executor.submit()`        | `CompletableFuture.supplyAsync()`         |
| ------ | -------------------------- | ----------------------------------------- |
| 接收任务   | `Callable<T>` 或 `Runnable` | `Supplier<T>`                             |
| 返回对象   | `Future<T>`                | `CompletableFuture<T>`                    |
| 执行线程池  | 指定的 `ExecutorService`      | 默认使用 `ForkJoinPool.commonPool()`（有特殊回退情况） |
| 获取结果   | `get()`                    | `get()`、`join()`                          |
| 后续任务编排 | 需要自行处理                     | 支持 `thenApply()` 等                        |
| 手动完成结果 | `Future` 接口不支持             | 支持 `complete()`                           |

默认使用`ForkJoinPool.commonPool()`，`supplyAsync()`也可以指定线程池，并且一般都建议指定线程池

```Java
ExecutorService executor =
    Executors.newFixedThreadPool(2);

CompletableFuture<Integer> future =
    CompletableFuture.supplyAsync(
        () -> {
            return 80;
        },
        executor
    );

Integer result = future.get();

executor.shutdown();
```

② `future.thenApply(...)`：当上一个 CompletableFuture 正常完成后，使用它的结果执行指定函数。 ^6e80a5

所以这里`future.thenApply(score -> score * 2);`相当于告诉 Java：任务 A 完成后，自动执行任务 B。

案例有两个不同的 Future：

```Java
future        // 保存 A 的结果：80
resultFuture  // 保存 B 的结果：160
```

`thenApply()`返回新 Future，不会修改原来的`Future`，这就是 CompletableFuture 的任务编排能力。

**“多个任务之间的依赖”，如下所示：**

```Plain
任务 A：查询语文成绩
任务 B：查询数学成绩
任务 C：计算两科总成绩
```

任务 C 必须等待 A 和 B 都完成。

这就叫任务依赖：

```Plain
任务 A ───┐
          ├──► 任务 C
任务 B ───┘
```

① 传统`Future`

通常需要自己获取两个任务的结果：

```Java
Future<Integer> futureA =
    executor.submit(() -> 80);

Future<Integer> futureB =
    executor.submit(() -> 90);

Integer a = futureA.get();
Integer b = futureB.get();

Integer total = a + b;
```

两个任务可以并发执行，但主线程需要等待和组合结果。

② `CompletableFuture`

可以使用：

```Java
CompletableFuture<Integer> futureA =
    CompletableFuture.supplyAsync(() -> 80);

CompletableFuture<Integer> futureB =
    CompletableFuture.supplyAsync(() -> 90);

CompletableFuture<Integer> totalFuture =
    futureA.thenCombine(
        futureB,
        (a, b) -> a + b
    );
```

> `.thenCombine(...)`：等 `futureA` 和 `futureB` 都正常完成之后，将两个结果交给 `(a, b) -> a + b` 进行处理。这里的 `a` 和 `b` 分别来自两个 Future 的结果。

最终：

```Java
Integer total = totalFuture.join();

System.out.println(total); // 170
```

`thenCombine()` 定义了任务之间的依赖关系，而不是让主线程逐个调用 `get()` 来协调。

## CompletableFuture

> 上一节已经引入了一些 `CompletableFuture` 相较于 `Future` 的优势，接下来将更系统地对 `CompletableFuture` 进行展开 。

`CompletableFuture<T>` 是一个类，它同时实现了两个接口：

```Java
public class CompletableFuture<T>
    implements Future<T>, CompletionStage<T>
```

| 接口                     | 主要职责                     |
| ---------------------- | ------------------------ |
| `Future<T>`            | 表示一个未来结果，允许查询、等待和取消      |
| `CompletionStage<T>`   | 定义任务完成后如何执行后续操作，以及如何组合任务 |
| `CompletableFuture<T>` | 提供上述能力的具体实现，并允许主动完成结果    |

在 [Future](<#^767848>) 类的时候，已经说明了，`Future`主要关注：

```Java
future.get();
```

而 `CompletableFuture` 还支持：

```Java
future.complete("结果");
```

以及：

```Java
future.thenApply(result -> result + "处理完成");
```

### 手动完成`CompletableFuture`

最简单的例子：

```Java
CompletableFuture<String> future =
    new CompletableFuture<>();
```

这里的泛型`<String>`表示 Future 正常完成后保存的结果类型是 `String`：

> 执行 `new CompletableFuture<>()` 不会创建新线程，也不会自动执行任何任务。它只是创建了一个尚未完成的 Future 对象。

**正常完成**：

```Java
future.complete("AI回答");
```

然后：

```Java
String result = future.get();
```

得到：

```Plain
AI回答
```

这里 `complete()` 有两个关键特性：

- 把结果保存到 Future 中，让等待的线程可以获取。
    
- Future 一旦正常完成或异常完成，后续普通的 `complete()` 不会覆盖已有结果。

`complete()` 的返回值是 `boolean`，成功让 Future 完成时返回 `true`；如果它已经完成，则返回 `false`。

> [对 complete() 进一步说明](<./CompletableFuture/对 complete() 进一步说明.md>)

**异常完成**：

例如：

```Java
CompletableFuture<String> future =
    new CompletableFuture<>();

future.completeExceptionally(
    new RuntimeException("AI调用失败")
);
```

此时调用：

```Java
future.get();
```

会抛出 `ExecutionException`，其 `cause` 是刚才传入的 `RuntimeException`。

> Future 完成了，不代表任务一定成功。任务可能正常完成，也可能异常完成。

### 异步执行任务

上述案例`new CompletableFuture<>();`本身不会启动任务。

Java 提供了方法`CompletableFuture.supllyAsync()`，可以异步运行，见下例：

```Java
CompletableFuture<String> future =
    CompletableFuture.supplyAsync(() -> {
        try {
            Thread.sleep(3000);
        } catch (InterruptedException ex) {
            Thread.currentThread().interrupt();
            throw new CompletionException(ex);
        }

        return "AI回答";
    });

System.out.println("主线程继续执行");

String result = future.join();

System.out.println(result);
```

默认情况下，`supplyAsync()` 使用 `ForkJoinPool.commonPool()` 执行任务；也可以显式传入自己的 `Executor`。执行流程如下：

```Plain
主线程
  │
  ├── supplyAsync(...)
  │       │
  │       └── 工作线程执行 lambda
  │                    │
  │                    └── return "AI回答"
  │
  ├── 主线程继续执行
  │
  └── future.join()
             │
             └── 获取异步计算结果
```

### 任务完成后执行下一步

这是 `CompletableFuture` 相比传统 `Future` 很重要的增强。

举个🌰：

```Java
CompletableFuture<Integer> future =
    CompletableFuture.supplyAsync(() -> 10)
        .thenApply(value -> value * 2)
        .thenApply(value -> value + 5);
```

执行逻辑：

```Plain
supplyAsync -> 10 -> thenApply -> 20 -> thenApply -> 25
```

最终：

```Java
future.join(); // 25
```

> `thenApply(...)`：前一个任务正常完成之后，拿到它的结果，再执行这个函数。[上一节](<#^6e80a5>)已经详细展开过，这里不再继续。
