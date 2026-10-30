# executor.submit()内部的 Lambda 详解

```Java
() -> {
    Thread.sleep(3000);
    return "AI回答";
}
```

相当于实现了：

```Java
Callable<String> task = () -> {
    Thread.sleep(3000);
    return "AI回答";
};
```

因为返回的是 `String`，所以：

```Java
Future<String> future = executor.submit(task);
```

这里 `Future` 的泛型就是 `String`。
