# FlightAction枚举类详解

^4553e2

```Java
public enum FlightAction {

    OWNER_NEW,

    OWNER_TAKEOVER,

    FOLLOWER_WAIT,

    REPLAY_SUCCESS,

    REPLAY_FAILURE
}
```

## 每个状态对应的行为

`FlightAction`里有五个状态，根据状态不同，会有不同的行为。

`OWNER_NEW`：表示当前 Key 第一次成功抢到 Owner。

`OWNER_TAKEOVER`：原来的 Owner 已经不能继续被信任，或者之前失败允许重新执行，现在我接管。

> `OWNER_NEW`和`OWNER_TAKEOVER`最终都是 Owner 路径，只是一个第一次 Owner，一个是接管上一代 Owner。

`FOLLOWER_WAIT`：已经有其他 Owner 在正常处理这个 Key，我不能再执行 AI。

`REPLAY_SUCCESS`：这个 Flight 以前已经成功完成。会尝试读取已经存在的成功结果。

> `FOLLOWER_WAIT`和`REPLAY_SUCCESS`都不会立即自己调用 AI，但是原因不同。
> `FOLLOWER_WAIT`是因为当前有`OWNER`正在处理，需要等待结果；
> `REPLAY_SUCCESS` 是已经成功了，已经有结果了，我可以直接读结果。

`REPLAY_FAILURE`：Redis 告诉当前请求，这个 Flight 已经处于当前不能正常继续执行的失败状态。