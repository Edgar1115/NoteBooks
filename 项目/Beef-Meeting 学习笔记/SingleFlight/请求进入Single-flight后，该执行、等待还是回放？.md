# 请求进入Single-flight后，该执行、等待还是回放？

## 单机版`InterviewAiSingleFlightService`

核心字段

```Java
private final ConcurrentMap<String, FlightEntry> flights =
    new ConcurrentHashMap<>();
```

> InterviewAiSingleFlightService由Spring Bean管理，同一JVM，多个请求访问的是同一个flights

> [什么是ConcurrentMap?](请求进入Single-flight后，该执行、等待还是回放？/什么是ConcurrentMap?.md)

> FlightEntry是
>
> ```Java
> private record FlightEntry(
>     CompletableFuture<Object> resultFuture, 
>     long expireAtMillis) {
>     }
> ```
>
> [CompletableFuture在这个项目的应用](请求进入Single-flight后，该执行、等待还是回放？/CompletableFuture在这个项目的应用.md)
> [什么是record](请求进入Single-flight后，该执行、等待还是回放？/什么是record.md)

### `execute()`收到什么？

方法：

```Java
public <T> T execute(

    // key：请求属于哪一个Flight
    String key,
    
    // 要执行的任务
    Supplier<T> supplier
)
```

#### 判断Single-flight是否启用

```Java
if (!Boolean.TRUE.equals(configuration.getEnable()) // getEnable()：判断单飞是否启用 
    || StrUtil.isBlank(key)  // 判断Key是否为空
    ) {
    // 如果两个判断条件都为真，直接执行
    
    meterRegistry
        .counter("ai_singleflight_miss_total")
        .increment();

    return supplier.get();
}
```

> 这里的`InterviewSingleFlightService`也启用了一个`MeterRegistry`，`AiCallGuardService`也启用了一个`MeterRegistry`，这里相当于是各自一个`MeterRegistry`吗？

#### 正常开始

```Java
AtomicBoolean newFlight =
    new AtomicBoolean(false);
```

> [AtomicBoolean是什么？为什么用AtomicBoolean?](请求进入Single-flight后，该执行、等待还是回放？/AtomicBoolean是什么？为什么用AtomicBoolean?.md)

用`newFlight`记录当前这一次`execute()`的调用，是否创建了新的Flight。

#### `flights.compute()`

```Java
FlightEntry entry =
    flights.compute(
        key,
        (ignored, existing) -> {

            if (existing == null
                || existing.expireAtMillis <= now) {

                newFlight.set(true);

                return new FlightEntry(
                    new CompletableFuture<>(),
                    now + ttlMillis
                );
            }

            return existing;
        }
    );
```

> [flights.compute()](请求进入Single-flight后，该执行、等待还是回放？/flights.compute().md)

调用:

```Java
flights.compute(K, lambda)
```

这个lambda会拿到：

```Java
existing
```

即当前Map有没有K对应的FlightEntry。

#### 不存在

```Java
existing == null
```

说明：

```Plain
当前 JVM
还没有人在处理 K
```

那么：

```Java
newFlight.set(true);
```

标记：

```Plain
“这个 Flight 是当前请求创建的。”
```

然后创建一个新的`FlightEntry`并放进Map:

```Java
new FlightEntry(
    new CompletableFuture<>(),
    now + ttlMillis
)
```

当前请求就成为`Leader`，或者分布式下的`Owner`。

##### 存在

如果

```Java
existing != null && existing.expireAtMillis > now
```

说明已经有一个有效Flight在处理K了，直接

```Java
return existing;
```

这个时候

```Java
newFlight == false
```

所以它知道自己不是Leader而是Waiter

#### Leader逻辑

进入Leader分支:

```Java
if (newFlight.get()) {
```

这里执行：

```Java
// 即第一节的() -> aiCallGuardService.execute(...)
T value = supplier.get();

// 广播，通知所有等待同一个Future的waiter已经拿到结果
entry.resultFuture.complete(value);

return value;
```

##### Leader执行失败逻辑

```Java
catch (Throwable ex) {

    entry.resultFuture
        .completeExceptionally(ex);

    // 把失败 Flight 从 Map 中删掉
    flights.remove(
        key,
        entry
    );

    throw ex;
}
```

> 这里的resultFuture.completeExceptionally()的作用？

#### Waiter逻辑

> meterRegistry有两个计数器
>
> 一个是"ai_singleflight_miss_total"，一个是"ai_singleflight_hit_total"
>
> 各自何意味？

之前的分支判断逻辑为"false"，则进入`Waiter`逻辑：

```Java
// 判断逻辑为 false
newFlight.get() == false

// 进入 Waiter 逻辑
T reused =
    (T) entry.resultFuture.get(
        resolveWaitTimeoutMillis(),
        TimeUnit.MILLISECONDS
    );
```

> 这里的resultFuter.get 为什么是传入时间呢？相当于是传入等待时间吗？

##### Waiter 等待超时

Waiter 设定了最多等`resolveWaitTimeoutMillis()`的时长，如果超时，触发超时逻辑：

```Java
// 删除 flghts 的 Entry
flights.remove(key, entry);

throw new CompletionException(
    new RejectedExecutionException(
    "single-flight wait timeout", ex)
);
```

这里只是 Waiter 等待超时，不等于原来的 Leader 停止。

> 这里对应的是 Waiter 等待的 Leader 的 Entry 吧？
