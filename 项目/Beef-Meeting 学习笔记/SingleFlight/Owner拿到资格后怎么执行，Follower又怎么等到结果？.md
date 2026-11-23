
# Owner拿到资格后怎么执行，Follower又怎么等到结果？

本节要讨论的流程图：

```Plain
executeDistributed()
    ↓
acquireOrJoin()
    ↓
得到 FlightAction
    │
    ├─ OWNER_NEW / OWNER_TAKEOVER
    │       ↓
    │   ownerExecute()
    │
    └─ FOLLOWER_WAIT
            ↓
        followerWait()
```

这一节是对 ownerExecute()和 followerWait()的展开讨论。

## 进入`ownerExecute()`

函数入口：

```Java
private String ownerExecute(
    String stage, // 当前 AI 业务阶段
    String requestKey, // 这个 Flight 的业务 Key
    Long ownerToken, // Redis 分给当前 Owner 的 Token
    Supplier<String> supplier, // () -> aiCallGuardService.execute() 
    StageFlightPolicy policy // 当前 stage 对应的 Single-flight 策略
 j)
```

> [supplier出处](<./业务请求如何进入Single-flight.md#^da1bae>)

先定义一个 Redis 运行态时间：
 
```Java
long runningTtlMillis = ... // 15秒
```

立即调用：

```Java
boolean markedRunning =
    flightCoordinatorRepository.markRunning(
        requestKey,
        nodeId(),
        ownerToken,
        runningTtlMillis  // 刚刚定义的 Redis 运行态时间
    ) ;
```

### 进入`markRunning`内部

注意，进入`markRunning`内部时，`FlightAction`为`OWNER_NEW`或`OWNER_TAKEOVER`，根据`ACQUIRE_OR_JOIN_SCRIPT_TEXT`，这两个状态的`status`都为`PENDING`。

`markRunning`内部有一个`Lua`脚本`MARK_RUNNING_SCRIPT_TEXT`，会对`Owner`再进行一次确认。

```Lua
-- 先比较 Redis 中的当前的 Owner
if redis.call('HGET', KEYS[1], 'ownerId') ~= ARGV[1]

-- 再比较 Redis 中的当前 OwnerToken
if redis.call('HGET', KEYS[1], 'ownerToken') ~= ARGV[2]

-- 比较 status
if status ~= 'PENDING' 

-- 将 status 从'PENDING'设置为'RUNNING'，updateAt 设置为 now，heartbeatAt 设置为now，过期时间设置为 now + runningTtlMillis，即前面的 runningTtlMillis，这里的过期时间是当前 flight 的过期时间
redis.call('HSET', KEYS[1], 'status', 'RUNNING', 'updatedAt', ARGV[3], 'heartbeatAt', ARGV[3], 'expireAt', ARGV[4])

-- 当前 Redis Key 的过期时间，同样是前面的 runningTtlMillis
redis.call('PEXPIRE', KEYS[1], tonumber(ARGV[5])
```

#### 为什么需要 `markRunning()` 再验一次 Owner？

举个🌰：

假设：

```Plain
t0：
A acquireOrJoin()
→ OWNER_NEW
→ token = 101
```

但是 A 因为线程调度、GC、网络卡顿等原因，停住了。过了一段时间，B 进来，Redis 判断 A 心跳失效：

```Plain
B
→ OWNER_TAKEOVER
→ token = 102
```

这时 Redis 的当前合法 Owner 已经是 B 了，但是 A 又恢复执行，如果 A 不检查，A 和 B 都会执行 AI，所以这里先对 A 做检查，就可以避免这种情况。

#### `markRunning()` 失败分支

`markRunning()`如果失败，执行以下逻辑：

```Java
if (!markedRunning) {
    return followerWait(
        stage,
        requestKey,
        policy,
        ...
    );
}
```

将当前 flight 的身份退化为 Follower，`followerWait`的逻辑见后续。

#### `markRunning()` 成功分支

构造`FlightOwnerContext`：

```Java
FlightOwnerContext ownerContext =
	FlightOwnerContext.builder().stage(stage).requestKey(
		requestKey
	).ownerId(nodeId()).ownerToken(ownerToken).policy(
		policy
	).build();
```

这个对象保存当前 Owner 执行期间需要的上下文：

```Plain
stage
requestKey
ownerId
ownerToken
policy
```

启动心跳任务：

```Java
String heartbeatTaskKey =
    flightHeartbeatManager.start(
        ownerContext,
        () -> flightCoordinatorRepository.heartbeat(...)
    );
```

---

`start()`逻辑：

入口：

```Java
public String start(
        FlightOwnerContext ownerContext,
        BooleanSupplier heartbeatAction)
        // 这里的 supplier 就是上面的.heartbeat()
```

先获取当前`policy`下的心跳执行间隔：

```Java
long intervalMillis = (...)
```

将`requestKey`和`OwnerToken`拼在一起，得到`taskKey`；

```Java
String taskKey = ownerContext.getRequestKey() + "|"
            + ownerContext.getOwnerToken();
```

**向定时线程池提交一个周期性任务，让线程池每隔** `intervalMillis` **毫秒执行一次心跳操作，并返回一个** `ScheduledFuture` **对象，用来控制这个任务。**

```Java
ScheduledFuture<?> future =
	scheduledExecutorService.scheduleAtFixedRate(
		() -> heartbeatAction.getAsBoolean(),
		intervalMillis,
		intervalMillis, // 每隔多久调度一次
		TimeUnit.MILLISECONDS
	)
```

> [ 本段代码详解](Owner拿到资格后怎么执行，Follower又怎么等到结果？/ScheduledFuture.scheduleAtFixedRate().md j)
> 
> 这段代码本质是`ScheduledExecutorService` **定时执行心跳，**`future` **是这个定时任务的控制句柄。**

然后将 `taskKey`和`future`压入`ConcurrentHashMap futures`里，并返回 `taskKey`由`haertbeatTaskKey`接收：
 
```Java
futures.put(
	taskKey,
	future
);

return taskKey;
```

---

`start()` 的含义是**注册一个周期心跳任务**，不是“同步把所有心跳执行完”。

整体关系如下：

```Plain
Owner 主线程
    ↓
start heartbeat
    ↓
继续 supplier.get()


调度线程
    ↓
每隔一段时间执行 heartbeat()
```

即两边为并行：

```Plain
主任务正在等 AI
          │
          ├──────────────┐
          ↓              ↓
     supplier.get()   heartbeat()
                     heartbeat()
                     heartbeat()
```

`Owner`心跳任务注册完成后，执行`supplier.get()`：

```Java
String result = supplier.get();
```

第一节里：

```Java
supplier =
    () -> aiCallGuardService.execute(
        stage,
        key,
        callable
    );
```

这里开始执行**之前延迟包装的 AI 任务**：

```Plain
AiCallGuardService.execute()
    ↓
第五节学习的保护链
    ↓
最终 callable.call()
    ↓
doChat()
    ↓
XingChenAIClient.chat()
```

### `supplier.get()`成功

假设 AI 返回：

```Java
String result // 例如评分结果字符串：{"score":85,"feedback":"..."}
```

Owner 接下来不会直接只`return result`，它要把结果保存下来让 Follower 能复用，整个顺序如下：

```Plain
supplier.get()
    ↓
serialize
    ↓
storeResult
    ↓
finishSuccess
    ↓
publish Stream
    ↓
L1 put
    ↓
return result
```

`serialize()`把 Java 中的 AI String 转换成“适合存 Redis 的结构”：

```Java
FlightStoredResult storedResult =
    flightResultSerializer.serialize(
        result,
        ownerToken,
        policy
    );
```

写 Redis：

```Java
flightCoordinatorRepository.storeResult( 
				requestKey,
                nodeId(),
                ownerToken,
                storedResult,
                resultTtlMillis)
```

注意，这里传入的是`requestKey`，之前我们一直用的是`metaKey`，两个 Key 存不同的结果：

```Plain
metaKey = Flight 元数据

resultKey = 真正 AI 结果
```

再次注意，每次在 redis 操作时，都会检查 `ownerToken`，这也是前面讲过的`Fencing Token`，主要是保护数据不被错误覆盖。

然后修改 `status`状态：

```Java
flightCoordinatorRepository.finishSuccess(
                // 更新status状态
                requestKey,
                nodeId(),
                ownerToken,
                resultTtlMillis
            
```

将`status`从`RUNNING`改为`SUCCEEDED`。同时还会写：

```Plain
resultRef
errorType = ""
errorCode = ""
retryable = 0
```

注意，这里的顺序不可以乱，一定要先`storeResult()` 再 `finishSuccess()`；

因为写 Redis 的过程中，可能发生异常，所以一定要写成功以后，再更新状态。

也就是说，在`resultKey`有结果，即 Redis 中已经写好结果，但是`status`更新之前，这时候存在一个中间态，这个时候不能说`resultKey`有结果，就说明 Flight 成功，一定要`status == SUCCEEDED`才行。

这里成功以后，向 Redis Stream 发送通知：

```Java
flightNotificationService.publish(...)
```

发送类似通知到 Redis Stream，通知等待这个 Flight 的 Follower，状态改变：

```Plain
owner_succeeded
SUCCEEDED
```

然后再将把结果写进 `Owner` 自己的 L1 Cache：

```Java
flightReplayLocalCache.put(
    stage,
    requestKey,
    result,
    policy
);
```

这样当前 JVM 以后再遇上同一个 Key 就不用再访问 Redis 了。

> 请注意，这里只在当前 JVM 本地，不能跨 JVM。

完成后，最后`return result`，至此，`Owner`的完整成功路径就结束了：

```Plain
AI 调用
↓
结果写 Redis
↓
状态 SUCCEEDED
↓
通知 Follower
↓
L1
↓
return
```

完整的方法路径：

```Plain
Owner
 ↓
markRunning
 ↓
start heartbeat
 ↓
supplier.get()
 ↓
serialize
 ↓
storeResult
 ↓
finishSuccess
 ↓
publish success
 ↓
L1 put
 ↓
return
```

### Owner 失败

如果`supplier.get()`抛异常，由于 Follower 还在等，所以 Owner 不能简单直接 throw，必须把`失败状态`也同步到 Redis。

先对异常进行分类：

```Java
// 异常归类
// err jorType
// errorCode
// retryable
FlightFailure failure = classifyFailure(ex);
```

然后通过`finishFailure`将`status`更改为`FAILED`：

```Java
flightCoordinatorRepository.finishFailure(
// finishiFailure 会调用 Lua 脚本，更新 redis 里的 status
                requestKey,
                nodeId(),
                ownerToken,
                failure.errorType,
                failure.errorCode,
                failure.retryable,
                positive(
                    policy.getFailedResultTtlMillis(),
                    60000L
                )
            );
```

注意，这里就在设置`retryable`了，第三节中的[Lua](多JVM怎么通过Redis安全决定Owner%20or%20Follower？.md#^b5e722)在这里设置。

这里设置了`status`后，同时也要向`Redis Stream`发送通知：

```Java
flightNotificationService.publish(...)
```

### 无论成功还是失败，都进行`finally`

`finally`会停止当前 JVM 里的 ScheduledFuture 心跳任务：

```Java
finally {
    flightHeartbeatManager.stop(
        heartbeatTaskKey
    );
}
```

---

## 进入 `followerWait()`

Redis 返回`FOLLOWER_WAIT`，进入：

```Java
followerWait(...)
```

`Follower`大致流程：

```Plaiin
先看看结果是不是已经成功了
        ↓
没有
        ↓
看看是不是已经失败了
        ↓
没有
        ↓
等待 Redis Stream
        ↓
同时轮询兜底
        ↓
状态发生变化
        ↓
重新读取
```

先尝试读一次结果：

```Java
String replay = tryReadSuccessReplay(
                stage,
                requestKey,
                policy
            );
```
---
> [tryReadSuccessReplay](<./Owner拿到资格后怎么执行，Follower又怎么等到结果？/tryReadSuccessReplay.md>) 逻辑
---

### 为什么先读一次结果？

因为存在竞态。例如：

```Plain
B 得到 FOLLOWER_WAIT
```

但就在 B 真正开始等待 Stream 之前：

```Plain
A 已经成功了
```

如果 B 直接开始等一个“新的 Stream 消息”，可能白等。所以 Follower 通常先：

```Plain
主动检查一次当前 Redis 状态/结果
```

如果已经成功，不用再等通知：

```Plain
直接回放
```


如果 `tryReadSuccessReplay` 失败，就进入等待流程：

```Java
flightNotificationService.waitForTerminalEvent(
                requestKey,
                // Math.min() 用来计算本次 Redis Stream 阻塞等待的最大时长。
                Math.min(
                // 单次 Redis Stream 阻塞读取允许的最大等待时间
                    streamBlockTimeoutMillis,
                // 当前 Follower 整体等待期限还剩多少毫秒
                    remainingMillis
                )
            );
```

注意，这里实际上什么都不干，这是一个**可以被 Redis 消息提前唤醒的阻塞等待机制**。

---
> [waitForTerminalEvent](<./Owner拿到资格后怎么执行，Follower又怎么等到结果？/waitForTerminalEvent.md>) 逻辑
---

然后再尝试通过`tryReadSuccessReplay()`读取。

### 为什么既用 Stream 又轮询？

假设只依赖`Redis Stream`，可能遇到：

```Plain
消息延迟
读取异常
订阅窗口错过
网络抖动
```

所以项目还设置了`poll fallback`，当前配置类似：

```Plain
stream-block-timeout-millis = 3000
poll-fallback-interval-millis = 2000
```

所以思路是：

```Plain
Stream
→ 快速通知

Poll
→ 兜底检查
```

这是一种：

> “事件通知 + 主动查询”的组合。

## 第四节完整双机故事

假设 A、B 同时请求 K。

第三节 Redis 决定：

```
A → OWNER_NEW token=101
B → FOLLOWER_WAIT
```

A：

```
ownerExecute()
    ↓
markRunning(101)
    ↓
RUNNING
    ↓
start heartbeat
    ↓
supplier.get()
    ↓
调用 AI
```

B：

```
followerWait()
    ↓
检查结果
    ↓
没有
    ↓
等待 Stream / Poll
```

A 调 AI 成功：

```
result
    ↓
serialize
    ↓
storeResult
    ↓
finishSuccess
        status=SUCCEEDED
    ↓
publish(owner_succeeded)
    ↓
A L1.put()
    ↓
return
```

B：

```
收到 Stream 通知
    ↓
重新读取
    ↓
meta = SUCCEEDED
    ↓
读取 resultKey
    ↓
deserialize
    ↓
得到 result
    ↓
B L1.put()
    ↓
return
```

最终：

```
A
→ 真正执行 AI

B
→ 没调用 AI
→ 复用 A 的结果
```

这就是前四节的完整闭环。