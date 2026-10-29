> 多个相同的 AI 请求同时到来时，尽量只让一个 Owner 真正执行 AI，其他请求等待或复用结果；即使系统有多个 JVM，也通过 Redis 协调 Owner 身份、接管、心跳和结果回放。

第一节 [业务请求如何进入Single-flight](%E4%B8%9A%E5%8A%A1%E8%AF%B7%E6%B1%82%E5%A6%82%E4%BD%95%E8%BF%9B%E5%85%A5Single-flight) 完成了:

```Plain
业务请求
   ↓
生成 Single-flight Key
   ↓
把真正任务包装成 Supplier
   ↓
交给 Single-flight
```

第二节 [请求进入Single-flight后，该执行、等待还是回放？](%E8%AF%B7%E6%B1%82%E8%BF%9B%E5%85%A5Single-flight%E5%90%8E%EF%BC%8C%E8%AF%A5%E6%89%A7%E8%A1%8C%E3%80%81%E7%AD%89%E5%BE%85%E8%BF%98%E6%98%AF%E5%9B%9E%E6%94%BE%EF%BC%9F)要做的就是：

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

第三节： [多JVM怎么通过Redis安全决定Owner or Follower？](<%E5%A4%9AJVM%E6%80%8E%E4%B9%88%E9%80%9A%E8%BF%87Redis%E5%AE%89%E5%85%A8%E5%86%B3%E5%AE%9AOwner or Follower%EF%BC%9F>)

第四节： [Owner拿到资格后怎么执行，Follower又怎么等到结果？](Owner%E6%8B%BF%E5%88%B0%E8%B5%84%E6%A0%BC%E5%90%8E%E6%80%8E%E4%B9%88%E6%89%A7%E8%A1%8C%EF%BC%8CFollower%E5%8F%88%E6%80%8E%E4%B9%88%E7%AD%89%E5%88%B0%E7%BB%93%E6%9E%9C%EF%BC%9F)