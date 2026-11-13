# 本地 L1 缓存 FlightReplayLocalCache 详解

## L1 Cache 的声明：

```Java
private final Map<String, CacheEntry> cache = new LinkedHashMap<>(128, 0.75F, true) {
	@Override
	protected boolean removeEldestEntry(Map.Entry<String, CacheEntry> eldest) {
		return size() > maxSize;
	}
};
```

### L1 Cache 代码段详解

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

### 什么是 `removeEldestEntry()`？

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

## L1的 [get()](请求进入Single-flight后，该执行、等待还是回放？.md#^ee47a8) 在做什么？

```Java
public synchronized String get(String stage, String requestKey) {
	// 这里就是前面拿到的 stage 和 requestKey
	String cacheKey = cacheKey(stage, requestKey);
	
	// 因为交给Spring Bean统一管理，所以是同一个HashMap
	CacheEntry entry = cache.get(cacheKey);
	
	// 下面的几个判断语句，都是判断当前JVM是否已经缓存了相同Key的成功结果
	if (entry == null) {  
		// 本地没有，直接返回null
		return null;
	}
	if (entry.expireAtMillis <= System.currentTimeMillis()) {
		// 本地有，但是过期了
		// 从 cache 中移除
		cache.remove( jcacheKey);
		// 然后返回null
		return null;
	}
	
	// 本地有且没有过期，直接返回
	return entry.value;
}
```