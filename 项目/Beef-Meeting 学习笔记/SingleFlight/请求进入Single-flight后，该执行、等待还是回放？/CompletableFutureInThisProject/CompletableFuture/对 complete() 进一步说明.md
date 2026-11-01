# 对 complete() 进一步说明

> 一个 `CompletableFuture` 只能从“未完成状态”变成“已完成状态”一次。只要它已经完成，无论是正常完成还是异常完成，再调用 `complete()` 都无法修改它的结果。

## 情况一：正常完成

```Java
CompletableFuture<Integer> future =
    new CompletableFuture<>();

// 第一次完成
boolean a = future.complete(100);

// 第二次尝试完成
boolean b = future.complete(200);

System.out.println(a);             // true
System.out.println(b);             // false
System.out.println(future.join()); // 100
```

> `join()`的功能与`get()`同。
> `CompletableFuture` 既能使用继承自 `Future` 接口契约的 `get()`，也能使用自己提供的 `join()`。日常使用 `CompletableFuture` 时，`join()` 通常更简洁，但需要特别注意异常处理和线程阻塞问题。

执行过程：

| 操作                          | 返回值     | Future 内部结果 |
| --------------------------- | ------- | ----------- |
| `new CompletableFuture<>()` | —       | 未完成         |
| `complete(100)`             | `true`  | 100         |
| `complete(200)`             | `false` | 仍然是 100     |
| `join()`                    | 100     | 100         |

- `complete(100)` 返回 `true`，说明这次调用成功将 Future 设置为完成状态。

- `complete(200)` 返回 `false`，说明 Future 已经完成，这次调用没有改变结果。

因此，最终结果依然是 `100`，而不是 `200`。

## 情况二：异常完成

```Java
CompletableFuture<Integer> future =
    new CompletableFuture<>();

// 第一次：异常完成
boolean a = future.completeExceptionally(
    new RuntimeException("AI 调用失败")
);

// 第二次：尝试正常完成
boolean b = future.complete(100);

System.out.println(a); // true
System.out.println(b); // false

System.out.println(future.join());
```

执行到 `join()` 时，会抛出 `CompletionException`，其原因是原来的 `RuntimeException("AI 调用失败")`。

这是因为：

1. `completeExceptionally(...)` 已经让 Future 进入异常完成状态。
    
2. 后续调用 `complete(100)`，无法将异常结果改成正常结果。
    
3. `join()` 仍然观察到之前的异常。

## 这样设计的原因

因为 `CompletableFuture` 可能被多个线程同时操作。

例如：

```Java
CompletableFuture<Integer> future =
    new CompletableFuture<>();

// 线程 A
new Thread(() -> {
    future.complete(100);
}).start();

// 线程 B
new Thread(() -> {
    future.complete(200);
}).start();
```

两个线程都尝试完成同一个 `CompletableFuture`。

最终只有一个线程能够成功完成它，另一个线程的 `complete()` 会返回 `false`。

至于最终结果是 `100` 还是 `200`，取决于哪个线程先成功完成 Future。

> `complete()` 不能覆盖已完成结果，但这不代表 `CompletableFuture` 的结果绝对不能被修改。
>
> `CompletableFuture` 还有一个特殊方法,可以强制覆盖已有结果：
>
> ```Java
> future.obtrudeValue(300);
> ```
>
> 这里不做展开，只做了解。
