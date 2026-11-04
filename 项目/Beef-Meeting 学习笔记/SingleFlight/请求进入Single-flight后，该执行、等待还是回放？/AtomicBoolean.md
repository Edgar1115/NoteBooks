# 什么是AtomicBoolean?

> `AtomicBoolean` 是 Java 并发编程中用于实现线程安全的布尔值操作的类。

最重要的作用：在多线程环境下，保证多个线程对同一个布尔状态的原子性操作。

## 普通的`boolean`

```Java
boolean running = false;
```

假设有两个线程，都希望判断 `running` 是否为 `false`，如果是，就将其改成 `true`。

```Java
if (!running) {
    running = true;
    System.out.println("执行任务");
}
```

^52a0ba

这里存在一个并发问题。两个线程同时执行时，可能出现的情况

| 顺序  | 线程 A                | 线程 B                |
| --- | ------------------- | ------------------- |
| 1   | 检查 running，发现 false | —                   |
| 2   | —                   | 检查 running，发现 false |
| 3   | 设置 running = true   | —                   |
| 4   | —                   | 设置 running = true   |
| 5   | 执行任务                | 执行任务                |

结果：两个线程都执行了任务。

问题在于[代码](AtomicBoolean.md#^52a0ba)包含两个独立操作：

1. 读取 `running` 的值。

2. 修改 `running` 的值。

这两个操作组合起来不是原子操作。线程 A 检查完 `running` 后，线程 B 完全可能在 A 修改之前插进来。也就是说，线程 B 本不应该执行任务，但是现在线程 B 却执行了任务。

## `AtomicBoolean`

```Java
AtomicBoolean running = new AtomicBoolean(false);

if (running.compareAndSet(false, true)) {
    System.out.println("执行任务");
}
```

> `running.compareAndSet(false, true)`：如果当前值是 false，就将它修改为 true，并返回 true；否则不修改，返回 false。

更重要的是，检查和修改是一个不可分割的[原子操作](ConcurrentHashMap.md#^6bdd87)。

```Plain
running = false 
线程 A -> CAS 成功: true -> 执行任务
(at the same time) 线程 B -> CAS 失败: false -> 不执行任务
```

这样可以保证，同一个初始为 false 的 AtomicBoolean，在没有其他线程重置它的情况下，只有一个线程能够成功执行这次 `compareAndSet(false, true)`。

## 在项目`Beef-Meeting`中的用法

```Java
AtomicBoolean newFlight = new AtomicBoolean(false);

FlightEntry entry = flights.compute(
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

if (newFlight.get()) {
    // Leader 执行实际 AI 调用
}
```

`AtomicBoolean newFlight` 主要是充当一个可修改的状态标记，让 Lambda 内部的判断结果能够传递到外部。

由于 Java 的 Lambda 不能直接修改它捕获的普通局部 `boolean` 变量，使用 `AtomicBoolean` 也顺便解决了这个问题。