# SingleFlight

> 多个相同的 AI 请求同时到来时，尽量只让一个 Owner 真正执行 AI，其他请求等待或复用结果；即使系统有多个 JVM，也通过 Redis 协调 Owner 身份、接管、心跳和结果回放。

第一节 [业务请求如何进入Single-flight](<./SingleFlight/业务请求如何进入Single-flight.md>) 完成了:

```Plain
业务请求
   ↓
生成 Single-flight Key
   ↓
把真正任务包装成 Supplier
   ↓
交给 Single-flight
```

第二节 [请求进入Single-flight后，该执行、等待还是回放？](<./SingleFlight/请求进入Single-flight后，该执行、等待还是回放？.md>)要做的就是：

> Single-flight 收到这个 `key + supplier` 以后，到底怎么决定“我自己执行”还是“等别人执行”？即确定Owner和Follower

分以下步骤：

```Plain
先理解单 JVM Single-flight
        ↓
再理解为什么单 JVM 不够
        ↓
进入 DistributedInterviewAiSingleFlightService
        ↓
L1 / Redis 协调 / FlightAction 分流
```

第三节： [多JVM怎么通过Redis安全决定Owner or Follower？](<./SingleFlight/多JVM怎么通过Redis安全决定Owner or Follower？.md>)

第四节： [Owner拿到资格后怎么执行，Follower又怎么等到结果？](<./SingleFlight/Owner拿到资格后怎么执行，Follower又怎么等到结果？.md>)