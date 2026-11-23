
# `tryReadSuccessReplay` 逻辑

先查本地 JVM的 L1 缓存，有就返回，没有继续：
 
```Java
String localReplay = flightReplayLocalCache.get(...)
```

如果 L1 缓存没有，就去 Redis 里查：

```Java
FlightStoredResult storedResult =
            flightCoordinatorRepository.getStoredResult(requestKey);
```
