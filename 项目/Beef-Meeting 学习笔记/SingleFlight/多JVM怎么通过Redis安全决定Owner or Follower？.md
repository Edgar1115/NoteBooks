---

---
# 多JVM怎么通过Redis安全决定Owner or Follower?


> 第二节我们已经知道，通过`FlightAcquireResult`拿到的结果，会告诉 Java 本次的状态。本节会拆解这个过程。

第三节要探究的完整过程：

```Plain
acquireOrJoin()
        ↓
Redis Lua
        ↓
读取 Flight 当前状态
        ↓
判断 Owner 是否还有效
        ↓
必要时生成新的 ownerToken
        ↓
原子修改 Redis
        ↓
返回 FlightAction
```

第二节我们已经学习了`FlightStatus`和`FlightAction`，第三节首先讨论一个枚举类`FlightStatus`，这个类是用来说明 Redis 中这个 Flight 当前处于什么生命周期状态：

```Java
public enum FlightStatus {
    PENDING,
    RUNNING,
    SUCCEEDED,
    FAILED,
    CANCELLED,
    EXPIRED
}
```

^5f980b

例如：

```Plain
PENDING = 已经有人取得 Owner 身份，但还没有正式进入运行

RUNNING = Owner 已确认身份，任务正在执行

SUCCEEDED = 已成功

FAILED = 已失败
```

> 第二节提到的`FlightAction`是指当前这一次新的请求应该干什么，不能搞混`FlightAction`和`FlightStatus`。

## `acquireOrJoin()`做什么？

接下来进入`FlightCoordinatorRepository.acquireOrJoin()`

`acquireOrJoin()`收到的参数：

```Java
public FlightAcquireResult acquireOrJoin(
    String stage,
    String requestKey,
    String ownerId,
    String sessionId,
    StageFlightPolicy policy
)
```

`ownerId`来源：

```Java
private static String resolveNodeId() {
    try {
	    // .getLocalHost()：获取当前主机的网络地址信息
        return InetAddress.getLocalH jost().getHostName()
            + "@"
			// 获取当前 JVM 的运行时管理对象
			// .getName()：获取当前 JVM 实例的运行时名称
            + Manageme jntFac jtory.getRuntimeMXBean().getName();
    } catch (UnknownHostException ex) {
        return ManagementFactory.getRuntimeMXBean().getName();
    }
}
```

`OwnerId` 可以理解为当前 Java 进程 / JVM 节点的身份。

这里注意一个点，`ownerId` 和 `ownerToken` 不是一个东西

```Plain
ownerId = 哪个 Java 节点

ownerToken = 第几代 Owner
```

## Owner Token来源

^1610f7

```Java
private static final String OWNER_SEQ_KEY =
    "ai:flight:owner-seq";
```

这里的`OWNER_SEQ_KEY`是 Redis Key。

## Redis 的两个 Key

```Java
List.of(
    metaKey(requestKey),
    OWNER_SEQ_KEY
)
```

`metaKey()`是在为 requestKey 做一个拼接：

```Java
private String metaKey(String requestKey) {
	// 结果：ai:flight:meta:<requestKey>
	return META_KEY_PREFIX + requestKey;
}
```

对两个 Key 做一个解释：

```Plain
KEYS[1]
= 当前 Flight 的元数据 Hash

根据后文代码，会存

status
stage
ownerId
ownerToken
requestKey
sessionId
heartbeatAt
expireAt
retryable
errorType
errorCode
...

-----------------------------------------

KEYS[2]
= Owner Token 自增计数器
```

^b5e722

## Lua 脚本

Lua 脚本要传入以下 8 个参数：

```Plain
--------------------------
以下为 acquireOrJoin()传入
--------------------------
ARGV[1] = stage [业务阶段]
ARGV[2] = ownerId [当前服务器节点标识]
ARGV[3] = requestKey [业务请求 key]
ARGV[4] = sessionId [面试会话 ID]

--------------------------
以下为默认配置
--------------------------
ARGV[5] = now
ARGV[6] = now + runningTtlMillis [预计到期时间]
ARGV[7] = runningTtlMillis [Redis 运行态 TTL，默认 15 秒]
ARGV[8] = takeoverDetectMillis [Owner心跳失活检测窗口，默认10 秒]
```

> Lua 脚本的优势
> 
> Redis 单条命令具有原子性，但是连续执行多条 Redis 命令，这几条命令整体不一定具有原子性
> 
> Redis 执行 Lua 脚本时，会将整个脚本作为一个原子执行单元。这样整体 Redis 命令就具有了原子性。

### Lua 脚本 `ACQUIRE_OR_JOIN_SCRIPT_TEXT` 详解

^dc833f

```Lua
-- 去当前 Flight 的元数据 Hash 里读取 status
local status =
    redis.call(
        'HGET',
        KEYS[1],
        'status'
    )
```

> `local` ：Lua 声明一个局部变量

根据前面传入的[requestKey](多JVM怎么通过Redis安全决定Owner%20or%20Follower？.md#^b5e722)，读取 status，即前面提到的[FlightStatus](多JVM怎么通过Redis安全决定Owner%20or%20Follower？.md#^5f980b)，读取到 status 后，Lua 从这里开始分流。

#### 分支一：Flight 不存在

```Lua
-- Redis 中没有读到这个 Flight 的 status
if not status then
```

即当前 Redis目前没有这个有效 Flight 记录。

```Lua
local token = redis.call('INCR', KEYS[2])
```

KEY[2]就是`OwnerToken`，从零开始自增，生成新 Token。

举例，`token = 101`，接下来：

```Lua
redis.call(
    'HSET',
    KEYS[1],

    'status', 'PENDING',
    'stage', ARGV[1],
    'ownerId', ARGV[2],
    'ownerToken', token,
    ...
)
```

这个时候 Redis 变成如下：

```Plain
【只列出关键字段】
status = PENDING
ownerId = JVM-A
ownerToken = 101
heartbeatAt = now
expireAt = now + runningTTL
retryable = 0
```

然后为 Key 设置过期时间：

```Lua
-- PEXPIRE，为 KEYS[1]设置过期时间，即前面的runningTtlMillis
-- 这个 Redis Key 的 TTL = 15 秒
redis.call(
    'PEXPIRE',
    KEYS[1],
    tonumber(ARGV[7])
)
```

到此该分支走完，返回：

```Lua
return 'OWNER_NEW|' .. token
```

得到：

```Plain
OWNER_NEW|101
```

`OWNER_NEW`为[FlightAction](请求进入Single-flight后，该执行、等待还是回放？.md#^17f33e),`101`为[ownerToken](多JVM怎么通过Redis安全决定Owner%20or%20Follower？.md#^1610f7)，所以这里返回的值，Java解析后会得到：

```Plain
action = OWNER_NEW
ownerToken = 101
```

这个分支的结果：**第一次创建 Flight 的请求成为新 Owner。**

> 注意，这里的`status`是`PENDING`而不是`RUNNING`。

#### 分支二：成功获得

```Lua
if status == 'SUCCEEDED' then
    return 'REPLAY_SUCCESS|' .. status
end
```

此时 Redis：`status = SUCCEEDED`，说明以前已经有人成功做完了。所以新请求不会重新抢 Owner，不会重新调用 AI，直接返回`REPLAY_SUCCESS|SUCCEEDED` + `status`。同分支一，Java 会得到：

```Plain
action = REPLAY_SUCCESS
status = SUCCEEDED
```

这个分支的结果：告诉 Java，“这个任务已经成功，可以去读 Result 了。”

#### 分支三：之前失败

```Lua
if status == 'FAILED' then
```

这里也有两种情况：

```Lua
local retryable = redis.call('HGET', KEYS[1], 'retryable') or '0'
```

Case ①：`retryable = 1`

这里说明之前失败了，但是允许重新执行，于是执行：

```Lua
local token = redis.call('INCR', KEYS[2])
```

生成新 Owner Token。例如`101` -> `102`，状态从`FAILED` -> `PENDING`。然后照常设置。

然后覆盖：

```Plain
ownerId = 当前新节点
ownerToken = 102
```

最后返回：

```Plain
OWNER_TAKEOVER|102
```

总结一下就是：

```Plain
FAILED + retryable=true
    ↓
新一代 Owner
    ↓
OWNER_TAKEOVER
```

这个分支结束，`status`为`PENDING`。

case ②：`retryable = 0`

这时不允许正常重跑，Lua 读取错误信息：`errorType`，`errorCode`；

```Lua
local errorType = redis.call('HGET', KEYS[1], 'errorType') or '' 
local errorCode = redis.call('HGET', KEYS[1], 'errorCode') or ''
```

并返回：`REPLAY_FAILURE|0|errorType|errorCode`。

举例：

```Plain
Key: ai:flight:meta:score-1001

Hash:
    status      → FAILED
    errorType   → TIMEOUT
    errorCode   → AI_TIMEOUT
```

#### 分支四：`CANCELLED / EXPIRED`

```Lua
if status == 'CANCELLED' or status == 'EXPIRED'
then
    return 'REPLAY_FAILURE|1|'
        .. status
        .. '|STATE'
end
```

这里暂时只需要知道这两个显式状态会被转换为 `REPLAY_FAILURE` Action。

#### 分支五：前面都不命中，就进入 Heartbeat 判断

前面四个分支都不进入，就进行 `Heartbeat` 判断，这时的`status`有两个状态：

```Plain
PENDING
RUNNING
```

先得到上一任`Owner`最近一次有效心跳时间：

```Lua
local heartbeatAt =
    tonumber(
        redis.call(
            'HGET',
            KEYS[1],
            'heartbeatAt'
        ) or '0'
    )
```

然后再次确认失活检测窗口：

```Lua
local takeoverDetectMillis = tonumber(ARGV[8])
```

然后开始判断`Owner`是否还健康：

```Lua
-- 当前时间 - 上一次心跳时间 <= 10 秒
if heartbeatAt > 0 and (now - heartbeatAt) <= takeoverDetectMillis
then
```

如果符合条件，就认为原 Owner 目前仍然健康，进入子分支

##### `Owner`健康

执行：

```Lua
HINCRBY followerCount 1
```

> followerCount 反映有多少次请求进入过这个 Follower 分支。

并返回`FOLLOWER_WAIT|原ownerToken`。

即：

```Plain
PENDING / RUNNING + 心跳仍健康
        ↓
FOLLOWER_WAIT
```

##### Heartbeat 已经超时

如果当前时间 - heartbeatAt = 12 秒，即超出失活检测窗口，Lua 执行超时逻辑。

此时原 Owner 被判断为失活，然后执行：

```Lua
-- 例如：101 → 102
local token = redis.call('INCR', KEYS[2])
```

此时 Redis 重新写，并刷新 TTL：

```Plain
status = PENDING
ownerId = 新节点 B
ownerToken = 102
heartbeatAt = 当前时间
```

此时`status`为`PENDING`，并返回：

```Plain
OWNER_TAKEOVER|102
```

### `ACQUIRE_OR_JOIN_SCRIPT_TEXT` 完整决策树

```Plain
读取 status
    │
    ├─ 不存在
    │    ↓
    │  INCR token
    │    ↓
    │  PENDING
    │    ↓
    │  OWNER_NEW
    │
    ├─ SUCCEEDED
    │    ↓
    │  REPLAY_SUCCESS
    │
    ├─ FAILED
    │    │
    │    ├─ retryable=1
    │    │     ↓
    │    │   新 token
    │    │     ↓
    │    │   OWNER_TAKEOVER
    │    │
    │    └─ retryable=0
    │          ↓
    │       REPLAY_FAILURE
    │
    ├─ CANCELLED / EXPIRED
    │    ↓
    │  REPLAY_FAILURE
    │
    └─ PENDING / RUNNING
         ↓
       检查 heartbeat
         │
         ├─ 健康
         │    ↓
         │ FOLLOWER_WAIT
         │
         └─ 失活
              ↓
            新 token
              ↓
            OWNER_TAKEOVER
```

### Heartbeat 超时与 Redis TTL 区分

```Plain
Heartbeat 超时
= Key 还可能存在
= 判断旧 Owner 是否失活

Redis TTL 到期
= 整个 Key 已经不存在
```

`heartbeat`会刷新 TTL

根据源码`HEARTBEAT_SCRIPT_TEXT`，先检查：

```Plain
ownerId
ownerToken
status == RUNNING
```

全部正确后，将心跳设置为当前时间：

```Lua
HSET heartbeatAt = now
```

并同时重新设置：

```Lua
PEXPIRE KEYS[1] runningTtlMillis
```

举个🌰：

正常 Owner 如果每 3 秒成功发一次心跳，每次都会重新：

```Plain
TTL = 15 秒
```

不是：

```Plain
15 + 15 + 15...
```

而是每次把“剩余 TTL”重新设成 15 秒。所以即使 AI 执行，20 秒，只要 Heartbeat 一直正常：

```Plain
Redis Key 不一定在第 15 秒过期
```

### Fencing Token

> Fencing Token（栅栏令牌）是一种通过单调递增的数字，防止过期的任务执行者继续修改共享资源的并发控制机制。

在本系统中，解决一个问题：旧 Owner 虽然已经失去了执行权，但它之前启动的任务可能还没有停止。如果它之后又尝试写入结果，就可能覆盖新 Owner 的正确结果。`ownerToken`就是起到这个作用。

举个🌰，有一个进程 A：

```Plain
A
ownerId = A
token = 101
```

A 开始执行 AI，后来 A 心跳停止，但是这时A 的 Java 线程或者已经发出去的 HTTP 请求不一定停止。12 秒后 B 来，这时 Redis 报：

```Plain
heartbeat 超时
```

这时 B 成为`Owner`：

```Plain
B
ownerToken = 102
OWNER_TAKEOVER
```

所以当前 Redis 合法 Owner 是：

```Plain
ownerId = B
ownerToken = 102
```

注意，A 虽然心跳停止，但是 A 的 Java 线程或者发出去的 HTTP 请求不一定停止，所以如果后来 A 的 AI 请求返回成功，这时 A 想：

```Java
storeResult(...)
```

但是 A 携带的信息是：

```Plain
ownerId = A
ownerToken = 101
```

这时脚本 `STORE_RESULT_SCRIPT_TEXT` 会进行判断：

```Lua
-- 判断 ownerId
if redis.call( 'HGET', KEYS[1], 'ownerId' ) ~= ARGV[1]
then
    return 0
end

-- 判断 ownerToken
if redis.call( 'HGET', KEYS[1], 'ownerToken' ) ~= ARGV[2]
then
    return 0
end
```

但是Redis 当前是`B / 102`，但是 A 带来的是`A / 101`，所以这里会校验失败，所以 A 的结果不能覆盖 B 这一代 Flight。

`FencingToken`主要是保护受 Lua 身份校验的 Redis 操作，比如：

```Plain
markRunning
heartbeat
storeResult
finishSuccess
finishFailure
```

可以确保旧`Owner`不能合法地更新当前 Flight，因为已经有了新的`Owner`。

注意，`FencingToken`是保护 Flight，但是旧`Owner`已经发出的第三方 AI HTTP 请求不能自动消失。

### Lua 脚本的结果拿到之后，做什么？

根据[前面](多JVM怎么通过Redis安全决定Owner%20or%20Follower？.md#^dc833f)提到的，Lua 脚本会根据不同的分支，返回不同的字符串内容，并最终交给`parseAcquireResult()`处理。

举一个🌰：`OWNER_NEW|101`

首先：

```Java
String[] parts = raw.split("\\|", -1);
```

得到：

```Plain
parts[0] = OWNER_NEW
parts[1] = 101
```

然后：

```Java
FlightAction action = FlightAction.from(parts[0]);
```

这里将字符串转化为对应的枚举常量。最后 Builder：

```Java
FlightAcquireResult.builder()
    .action(action)
    .ownerToken(101L)
    .build();
```

最终 Java得到：

```Plain
FlightAcquireResult

action = OWNER_NEW
ownerToken = 101
```

然后回到第二节`DistributedInterviewAiSingleFlightService.executeDistributed()`里的`switch (acquireResult.getAction())`，到这里就完成了闭环。

第三节的核心是：**Redis 如何原子地决定这次请求到底是哪一种 Action，以及怎么通过 Heartbeat + Owner Token 防止旧 Owner 继续污染共享状态。**
