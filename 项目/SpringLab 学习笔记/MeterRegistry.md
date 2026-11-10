# MeterRegistry

`MeterRegistry`通过「指标名称（Name）」和「标签（Tags）」来区分不同的监控数据。 ^430ec5

用法 ①：

类`InterviewAiSingleFlightService`：

```Java
// 用法第一处
meterRegistry.counter(
    "ai_singleflight_hit_total"
).increment();

// 用法第二处
meterRegistry.counter(
    "ai_singleflight_miss_total"
).increment();
```

这里通过指标名称，相当于注册了两个不同的 Counter（计数器）指标。

过程解释：

```Plain
- 根据指标名称（和 Tags ）查找已有 Counter。
    
- 如果不存在，就创建并注册一个 Counter。
    
- 如果已经存在，就返回已有的 Counter。
    
- .increment() 将对应 Counter 的值加 1。
```

用法 ②：

类`AiCallGuardService`：

```Java
meterRegistry.gauge(
    "ai_circuit_state",
    Tags.of("stage", stage),
    gaugeTarget
);
```

|参数|含义|
|---|---|
|`"ai_circuit_state"`|指标名称（Metric Name）|
|`Tags.of("stage", stage)`|指标标签（Tags）|
|`gaugeTarget`|被监控的对象，其数值代表熔断器当前状态|

这里是同一个指标名称（Metric Name）下，可以通过不同的标签（Tags）区分多个独立的计数器（Counter）。