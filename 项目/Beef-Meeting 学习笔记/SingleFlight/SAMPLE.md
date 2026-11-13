# ONE

^8fa06e

> 用两个 JVM 走一遍分布式逻辑

存在两个 JVM

```Plain
JVM A
JVM B
```

同时收到完全相同评分请求：

```Plain
requestKey = K
```

A : 

```Plain
executeDistributed(K)
    ↓
A 的 L1
    ↓
没有 K
    ↓
acquireOrJoin(K)
```

Redis 最终告诉 A：

```Plain
OWNER_NEW
```

所以 A -> ownerExecute();

---

同时 B 也进来：

```Plain
executeDistributed(K)
    ↓
B 的 L1
    ↓
没有 K
```

注意：A 的 L1 和 B 的 L1 是两块不同 JVM 内存。所以即使 A 本机之后有缓存，B 此刻也看不到。

这时 B：

```Plain
acquireOrJoin(K)
```

由于 Redis 是共享的，所以：

```Plain
这个 K 已经有人处理
```

Redis 返回`FOLLOWER_WAIT`；

这时 B -> followerWait()，不调用 AI。

两个 JVM 实现如下流程：

```Plain
JVM A → Owner → 调 AI

JVM B → Follower → 等待
```

Redis 解决了单机 `ConcurrentHashMap` 无法跨 JVM 的问题。

---

上面的逻辑是 A 没有完成，如果 A 已经完成 K，并把成功结果写进共享 Redis。

后来 B 第一次收到 K：

```Plain
自己的 L1
    ↓
没有 K
```

这里的 L1 是本地缓存，B 中 L1 是没有结果的，结果在共享 Redis 里。

此时 B -> acquireOrJoin(K); Redis 可能返回`REPLAY_SUCCESS`;

所以 B：

```Plain
tryReadSuccessReplay()
    ↓
读取共享 Redis 结果
    ↓
返回
```

这时 B 拿到结果后，就可以把结果写进自己的 L1，这样下次 B 再收到 K，就可以直接在本地 L1 中拿到结果，不用访问 Redis。