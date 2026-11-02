# 什么是ConcurrentHashMap?

> `ConcurrentHashMap` 是 Java 提供的一个线程安全的哈希表（Hash Table），主要用于在多线程环境下安全地存储和访问键值对（Key-Value）。

`ConcurrentHashMap` = 支持多个线程并发读写的 `HashMap`。

## HashMap

`Map`的主要作用是存储键值对。

```Java
Map<String, Integer> map = new HashMap<>();

map.put("张三", 100);
map.put("李四", 200);

System.out.println(map.get("张三")); // 100
```

- `String`：Key 的类型。
    
- `Integer`：Value 的类型。
    
- `put()`：添加或者更新键值对。
    
- `get()`：根据 Key 获取 Value。

`HashMap`的问题：它不是线程安全的。

举个🌰(①)，假设有两个线程，同时操作一个`HashMap`：

```Java
Map<String, Integer> map = new HashMap<>();

// 线程 A
map.put("张三", 100);

// 线程 B
map.put("李四", 200);
```

由于两个线程可能同时修改内部数据结构，`HashMap` 无法保证并发操作的正确性。

这就引出了 `ConcurrentHashMap`。

## ConcurrentHashMap解决了什么问题？

上述案例①：

```Java
Map<String, Integer> map = new ConcurrentHashMap<>();

// 线程 A
map.put("张三", 100);

// 线程 B
map.put("李四", 200);
```

现在两个线程可以安全地并发执行 `put()`。

> 并发：多个任务在时间上存在重叠，执行过程可以相互交错，但不一定在同一时刻执行。
> 
> 并行：多个任务在同一时刻真正同时执行，通常需要多个 CPU 核心。

### ConcurrentHashMap 怎么保证线程安全？

这里涉及到 Java 并发编程的一个核心概念：原子性（Atomicity）。

> 原子性：一个操作在并发环境中具有不可分割的特性。即一个操作即使在内部由多个步骤组成，对其他并发线程而言，也应该表现为一个不可分割的整体。
> 
> 原子性三个关键特征：
> 
>- 不可分割性：从其他线程的观察角度看，整个操作是一个整体。
> 
>- 不可交错性：其他线程不能在该操作执行到一半时，介入并影响它的中间状态。
>    
>- 状态一致性：其他线程不会观察到该原子操作产生的部分更新结果。

举个🌰(②)：

```Java
map.put("张三", 100);
```

`ConcurrentHashMap` 可以保证这一次 `put()` 操作的线程安全。普通的`HashMap`不能保证线程安全。

但是如果写：

```Java
if (!map.containsKey("张三")) {
    map.put("张三", 100);
}
```

^b04ed4

这里会出现问题，因为这里是两个独立的操作。

假设线程 A，线程 B 同时执行：

|时间|线程 A|线程 B|
|---|---|---|
|T1|`containsKey()` → false||
|T2||`containsKey()` → false|
|T3|`put("张三", 100)`||
|T4||`put("张三", 100)`|

结果两个线程都认为自己是第一个插入数据的线程。虽然 `containsKey()` 和 `put()` 各自是线程安全的，但两个操作组合起来，不一定具有原子性。

这是可以使用：

```Java
map.putIfAbsent("张三", 100);
```

它的含义是：当 Key 不存在时插入数据，否则保留原值。检查 Key 是否存在以及插入数据，在这里属于一次原子操作。功能同[这一段代码](ConcurrentHashMap.md#^b04ed4)，但是具有原子性。

一个线程安全的🌰(③)：

```Java
ConcurrentHashMap<String, Integer> map =
    new ConcurrentHashMap<>();

// 线程 A
Thread threadA = new Thread(() -> {
    map.put("A", 100);
});

// 线程 B
Thread threadB = new Thread(() -> {
    map.put("B", 200);
});

threadA.start();
threadB.start();

threadA.join();
threadB.join();

System.out.println(map);
```

两个线程同时向同一个 Map 写入数据。

最终一定能够得到包含以下两个键值对的 Map：

```Plain Text
A = 100
B = 200
```

顺序不一定相同。

这就是 `ConcurrentHashMap` 线程安全的一个体现：即使多个线程同时修改 Map，它也能保证这些操作不会破坏 Map 内部的数据结构。

### `HashMap`为什么会线程不安全？

`HashMap` 底层主要使用数组 + 链表 + 红黑树存储数据（以 JDK 8 及以后版本为例）。

举个🌰(④)：

```Java
Map<String, Integer> map = new HashMap<>();

map.put("A", 100);
map.put("B", 200);
```

插入数据时，`HashMap` 会根据 Key 的哈希值，计算它应该存放在哪个数组位置。

假设 `"A"` 和 `"B"` 恰好被分配到同一个数组位置（Hash 冲突）。

正常情况下，两个节点会被正确连接起来。

```Plain Text
table[0]
table[1] -> A = 100 -> B = 200
table[2]

 示意图：两个 Key 的哈希值映射到同一个桶，节点通过链表连接。
```

如果两个线程同时插入，此时`table[1]`为空。

线程 A 执行：

```Java
map.put("A", 100);
```

线程 B 执行：

```Java
map.put("B", 200);
```

为了便于理解，将`HashMap`内部的部分插入逻辑简化为：

```Java
if (table[index] == null) {
    table[index] = new Node(key, value);
}
```

> 这只是帮助理解的简化代码，并不是 `HashMap` 完整源码。

现在可能发生下面的执行顺序：

|时间|线程 A|线程 B|
|---|---|---|
|T1|发现 `table[1] == null`||
|T2||发现 `table[1] == null`|
|T3|插入 A 节点||
|T4||插入 B 节点，覆盖数组位置上的 A 节点引用|

最终的可能结果：

```Plain
// T3: 线程 A 插入成功
table[1] -> A = j 100

// T4: 线程 B 覆盖该位置引用
table[1] -> B = 200
```

这个时候再执行：

```Java
System.out.println(map.get("A")); // 可能是 null
System.out.println(map.get("B")); // 200
```

线程 A 明明执行过 `put("A", 100)`，但它的数据却丢失了。

问题的根本原因是：`HashMap` 没有保证两个线程对同一个桶的并发修改是原子的。

注意，这里的[操作](ConcurrentHashMap.md#^b04ed4)，虽然用的是 `ConcurrentHashMap`，但是同样不能保证线程安全，但是`ConcurrentHashMap`提供了将两个步骤合并成一个的原子操作`putIfAbsent()`，这里就会保证操作的原子性。