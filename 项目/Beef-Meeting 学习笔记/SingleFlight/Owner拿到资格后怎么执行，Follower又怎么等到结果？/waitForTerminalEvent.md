# `waitForTerminalEvent` 逻辑

从指定的 Redis Stream 中读取最多一条新消息。如果暂时没有新消息，就阻塞等待，直到新消息到达或等待超时：

```Java
// 两个 Object 分别对应 Redis Stream 消息中的字段名和字段值。
List<MapRecord<String, Object, Object>> records =
    stringRedisTemplate.opsForStream().read(
    // StreamReadOptions：决定怎么读取 
    // empty()：创建一个初始没有额外读取限制的选项对象
        StreamReadOptions.empty()
    // count(1)：本次读取最多返回 1 条消息
            .count(1)
    // block()：设置 Redis Stream 的阻塞等待时间
            .block k(
            // Redis 最多阻塞等待 3 秒
                Duration.ofMillis(
                    Math.max(1L, blockTimeoutMillis)
                )
            ),
	// StreamOffset：决定从哪个 Redis Stream、哪个位置开始读取
        StreamOffset.create(
		// streamKey()：指定要读取的 Redis Stream Key
            streamKey(requestKey),
		// ReadOffset.latest()：从 Redis Stream 当前的末尾开始，等待后续新产生的消息，也就是说当前代码开始执行后，从现在开始监听新消息，之前的旧消息就不读
            ReadOffset.latest()
        )
    );
```

两个 Object，见`FlightNotificationService.publish()`：

```Java
Map<String, String> body = new HashMap<>();

body.put("eventType", "owner_succeeded");
body.put("status", "SUCCEEDED");
body.put("ownerToken", "101");
body.put("retryable", "0");
```

`MapRecord`结构，举例如下：

```Plain
Stream Key: ai:flight:stream:1001

消息内容:
    eventType  = owner_succeeded
    status     = SUCCEEDED
    ownerToken = 101
    retryable  = 0
```

请注意，`waitForTerminalEvent` 虽然有返回值，但是`FollowerWait()`没有接收，**在这里主要利用的是它的阻塞等待能力**
