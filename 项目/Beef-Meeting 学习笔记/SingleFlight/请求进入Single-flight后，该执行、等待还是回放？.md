# 请求进入Single-flight后，该执行、等待还是回放？

## 单机版`InterviewAiSingleFlightService`

核心字段

```Java
private final ConcurrentMap<String, FlightEntry> flights =
    new ConcurrentHashMap<>();
```

> InterviewAiSingleFlightService由Spring Bean管理，同一JVM，多个请求访问的是同一个flights

> [什么是 ConcurrentHashMap?](<./请求进入Single-flight后，该执行、等待还是回放？/ConcurrentHashMap.md>)

> FlightEntry是
>
> ```Java
> private record FlightEntry(
>     CompletableFuture<Object> resultFuture, 
>     long expireAtMillis) {
>     }
> ```
>
>[CompletableFuture在这个项目中的应用](<./请求进入Single-flight后，该执行、等待还是回放？/CompletableFutureInThisProject.md>)
>
>[什么是record](<./请求进入Single-flight后，该执行、等待还是回放？/什么是record.md>)

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

> Q : 这里的`InterviewSingleFlightService`也启用了一个[MeterRegistry](<../../SpringLab 学习笔记/MeterRegistry.md>)，`AiCallGuardService`也启用了一个`MeterRegistry`，这里相当于是各自一个`MeterRegistry`吗？
> 
> A : 不是。详见[MeterRegistry](../../SpringLab%20学习笔记/MeterRegistry.md#^430ec5)

#### 正常开始

```Java
AtomicBoolean newFlight =
    new AtomicBoolean(false);
```

> [什么是 AtomicBoolean?](<./请求进入Single-flight后，该执行、等待还是回放？/AtomicBoolean.md>)
> 
> [这里为什么用 AtomicBoolean 而不用普通的 Boolean](请求进入Single-flight后，该执行、等待还是回放？/AtomicBoolean.md#^63813c)

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

> [.compute()的用法](请求进入Single-flight后，该执行、等待还是回放？/ConcurrentHashMap.md#^7395d1)
> 
> 这里就是，如果当前 Flight 不存在，创建一个新的 Flight，并设置前面的 atomicBoolean newFlight 为 true，表示这个 Flight 是当前请求创建。即标记当前为 Leader；
> 
> 如果 Flight 已经存在，就直接返回，并且 atomicBoolean newFlight 为 false，表示当前为 Waiter；
> 
> 详细流程继续向下看。

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

> Q : 这里的resultFuture.completeExceptionally()的作用？
> 
> A : 将一个尚未完成的 `CompletableFuture` 标记为异常完成（Exceptional Completion）。

#### Waiter逻辑

> meterRegistry有两个计数器
>
> 一个是"ai_singleflight_miss_total"，一个是"ai_singleflight_hit_total"
>
> 各自何意味？
> 
> `ai_singleflight_miss_total`（未命中次数）：当前请求没有复用已有的 Flight，需要自己调用 `supplier.get()` 执行真实业务。
> 
> `ai_singleflight_hit_total`（命中次数）：当前请求复用了已有的 Flight，不需要自己执行真实业务，而是等待并获取已有 Flight 的执行结果。
> 
> 这里的命中（Hit）和未命中（Miss），指的是：当前请求能不能找到一个可以复用的 Flight。

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

> Q : 这里的resultFuter.get 为什么是传入时间呢？相当于是传入等待时间吗
> 
> A : 是的，这里传入的是最大等待时间，即超时时间。

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

> Q : 这里对应的是 Waiter 等待的 Leader 的 Entry 吧？
> 
> A : 是的。这里是等待超时的情况，会移除掉`key`对应的`entry`。

整体流程大致为：

```Plain
① Leader 第一次进入 compute()，创建新的 FlightEntry，并放入 flights。
    
② Follower 随后进入 compute()，发现 existing 已存在且没有过期，直接返回 existing。
    
③ 因此，Follower 的局部变量 entry，与 Leader 创建的 FlightEntry，引用的是同一个对象。
    
④ Follower 调用 entry.resultFuture.get() 等待 Leader 完成。
    
⑤ 如果等待超时，执行 flights.remove(key, entry);
```

> Q：正如前面所提到的，waiter 等待超时，不等于原来的 leader 停止，那么发生这种情况会怎么处理？
>
> A：如果 Waiter 超时删除了旧的 Flight，而 Leader 还没有执行完，那么新的请求进来时，可能发现 `flights` 中没有对应的 key。于是它会创建新的 Flight，成为新的 Leader，再次调用 AI。这样可能出现：
>
> - Leader A：仍在执行第一次 AI 调用。
  >  
> - Leader B：开始执行第二次 AI 调用。
>    
> - 两个 Leader 同时处理相同 key 的业务。
   > 
   > 这意味着 SingleFlight 并不能在所有情况下保证同一个 key 始终只有一个正在执行的任务。在等待超时、Flight 过期等情况下，可能出现重复执行。

## 分布式 Single-flight

入口类：`DistributedInterviewAiSingleFlightService.executeDistributed()`

实际代码入口：

```Java
// DistributedInterviewAiSingleFlightService.execute()
return executeDistributed(
    stage,
    requestKey,
    supplier
);
```

常规整理 stage，requestKey，policy

```Java
// 整理 stage
String safeStage =
    StrUtil.blankToDefault(
        stage,
        "interview-default"
    );

// 整理 requestKey
String safeRequestKey =
    StrUtil.blankToDefault(
        requestKey,
        safeStage + "|no-key"
    );

// 整理 policy
StageFlightPolicy policy =
    configuration.resolveStagePolicy(
        safeStage
    );
```

先检查本地缓存，如果拿到 localReplay，直接返回：
 j
```Java
// 检查本地缓存
String localReplay =
    flightReplayLocalCache.get(
        safeStage,
        safeRequestKey
    );
	
// 根据 RequestKey，如果从 L1 本地结果缓存拿到结果，直接返回结果
if (localReplay != null) {

	return localReplay;
}
```

查本地缓存的具体流程，不访问 redis：

```Plain
请求进入 Distributed Single-flight
        ↓
先查当前 JVM L1
        ↓
命中
        ↓
直接返回
```

### 本地 L1 缓存`FlightReplayLocalCache`

```Java
private final Map<String, CacheEntry> cache = new LinkedHashMap<>(128, 0.75F, true) {
	@Override
	protected boolean removeEldestEntry(Map.Entry<String, CacheEntry> eldest) {
		return size() > maxSize;
	}
};
```

```Java
new LinkedHashMap<>(128, 0.75F, true)
```

```Plain
128 : 初始容量，代表 Hash 表底层桶数组的长度，每个数组位置为一个桶，即有 128 个桶

0.75F : 负载因子，用于控制扩容阈值，Hash 表在存储多少元素后需要扩容
扩容阈值 = 容量 * 负载因子
以这里举例，初始容量为 128，负载因子为 0.75，那么扩展阈值即为 96，当插入 1~96 个元素时，不扩容，插入第 97 个元素时，即超过阈值，触发扩容，扩容一般是底层桶数组的长度翻倍。
在负载因子为 0.75 的情况下，Hash 表通常不会等到所有桶都被占用才扩容。

true : 按访问顺序排序，每次访问一个已有元素，就把它放到队伍最后面。举例：插入时顺序为 A -> B -> C，执行 get("A")，顺序就变成 B -> C -> A
```

```Java
@Override
protected boolean removeEldestEntry(
    Map.Entry<String, CacheEntry> eldest
) {
    return size() > maxSize;
}
```

什么是 `removeEldestEntry()`？

方法声明：

```Java
// 原始声明
protected boolean removeEldestEntry(
    Map.Entry<K, V> eldest
)
```

默认实现返回 `false`，也就是说，普通的 `LinkedHashMap` 不会因为元素数量过多而自动删除元素。

通过`@Override`重写之后，作用为：每次向 `LinkedHashMap` 插入新元素时，判断缓存中的元素数量是否超过 `maxSize`，如果超过，就自动删除最旧的元素。

注意：这个方法不是你主动调用的，而是 `LinkedHashMap` 在插入新元素后自动调用的。

