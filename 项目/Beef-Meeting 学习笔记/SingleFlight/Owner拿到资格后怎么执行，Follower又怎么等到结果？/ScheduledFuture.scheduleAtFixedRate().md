# .scheduleAtFixedRate()代码详解

```Java
ScheduledFuture<?> future =
	scheduledExecutorService.scheduleAtFixedRate(
		() -> heartbeatAction.getAsBoolean(),
		intervalMillis,
		intervalMillis, // 每隔多久调度一次
		TimeUnit.MILLISECONDS
	)
```

`scheduledExecutorService` 是一个 `ScheduledExecutorService` 对象。

它的 `scheduleAtFixedRate()` 方法声明如下：

```Java
public ScheduledFuture<?> scheduleAtFixedRate(
    Runnable command,
    long initialDelay,
    long period,
    TimeUnit unit
);
```

这个方法接收四个参数：

| 参数             | 类型         | 作用            |
| -------------- | ---------- | ------------- |
| `command`      | `Runnable` | 要执行的任务        |
| `initialDelay` | `long`     | 第一次执行前等待多久    |
| `period`       | `long`     | 两次任务计划开始时间的间隔 |
| `unit`         | `TimeUnit` | 时间单位          |

`Runnable`函数式接口，只有一个抽象方法 `run()`，没有返回值。：

```Java
@FunctionalInterface
public interface Runnable {
    void run();
}
```

因此：

```Java
() -> heartbeatAction.getAsBoolean()
```

相当于：

```Java
new Runnable() {
    @Override
    public void run() { // void，无需 return
        heartbeatAction.getAsBoolean();
    }
}
```

这里的`heartbeatAction.getAsBoolean()`到底在执行什么?

`heartbeatAction`是从`DistributedInterviewAiSingleFlightService`传入的：

```Java
String heartbeatTaskKey = flightHeartbeatManager.start(
    ownerContext,
    () -> flightCoordinatorRepository.heartbeat( // 这就是传入的heartbeatAction
        requestKey,
        nodeId(),
        ownerToken,
        runningTtlMillis
    )
);
```

`hearbeat()`内部主要执行：

```Java
Long result = stringRedisTemplate.execute(
    HEARTBEAT_SCRIPT,
    // Redis Key 和 Lua 脚本参数
);

return Long.valueOf(1L).equals(result);
```

---

`HEARTBEAT_SCRIPT` 内部执行：

这个脚本主要是**更新心跳记录并延长 Flight 元数据 Key 的 TTL。**

先进行一些例行检查：

```Lua
-- 检查 OwnerId 是否一致
if redis.call('HG jET', KEYS[1], 'ownerId') ~= ARGV[1]

-- 检查 OwnerToken 是否一致
if redis.call('HGET', KEYS[1], 'ownerToken') ~= ARGV[2]

-- 检查 Flight 是否处于 RUNNING 状态
if redis.call('HGET', KEYS[1], 'status') ~= 'RUNNING'
```

更新心跳字段：

```Lua
-- 更新心跳相关字段
redis.call(
    'HSET',
    KEYS[1],
    'updatedAt', ARGV[3], --将最近一次更新 Flight 的时间更新为 now
    'heartbeatAt', ARGV[3], -- 将最近一次成功心跳的时间更新为 now
    'expireAt', ARGV[4], -- 预期过期时间设置为 now + runningTtlMillis
    'lastHeartbeatNode', ARGV[1] -- 更新本次心跳为当前节点
)

-- 刷新 Redis Key 的 TTL
redis.call('PEXPIRE', KEYS[1], tonumber(ARGV[5]))
```

---

注意，`scheduleAtFixedRate()` 是**固定频率调度**。例如任务原本安排在第 3、6、9 秒开始执行。如果第 3 秒的任务执行了 4 秒，直到第 7 秒才结束，那么下一次可能马上开始，但同一个周期性任务不会并发重叠执行。