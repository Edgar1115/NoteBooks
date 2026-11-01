# CompletableFuture在这个项目的应用

>[CompletableFuture详解 ](<CompletableFutureInThisProject/CompletableFuture.md>)

① 手动创建一个未完成的 Future

```Java
return new FlightEntry(
    new CompletableFuture<>(),
    now + ttlMillis
);
```

② leader 主动完成 Future:

```Java
T value = supplier.get();

entry.resultFuture.complete(value);
```

③ follower 等待 Future 的结果：

```Java
T reused = (T) entry.resultFuture.get(
    resolveWaitTimeoutMillis(),
    TimeUnit.MILLISECONDS
);
```

本地 SingleFlight 没有使用 `CompletableFuture` 的异步任务执行能力，而是使用了它的结果共享和线程等待/通知能力。
