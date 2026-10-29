# 为什么这里是 task -> new Thread(task, "ai-worker")

`ThreadFactory`是一个函数式接口：

```Java
@FunctionalInterface
public interface ThreadFactory {

    Thread newThread(Runnable r);

}
```
这个方法明确规定：

- 参数类型是 `Runnable`
- 返回值类型是 `Thread`
因此：

```Java
task -> new Thread(task, "ai-worker")
```

实际上可以写成：

```Java
(Runnable task) -> {
    return new Thread(task, "ai-worker");
}
```

```Java
Executors.newSingleThreadExecutor(...)
```
一个创建单线程线程池的方法

```Java
task -> new Thread(task, "ai-worker")
```
传进去的是一个 Lambda 表达式，相当于实现了
```Java
public Thread newThread(Runnable task) {
    return new Thread(task, "ai-worker");
}
```
> 线程池给我一个 `Runnable task`，我就根据这个 `task` 创建一个名为 `ai-worker` 的线程，并把线程对象返回给线程池。

```Java
ExecutorService executor =
    Executors.newSingleThreadExecutor(
        task -> new Thread(task, "ai-worker")
    );
```
`newSingleThreadExecutor()` 返回创建好的线程池对象，我们用 `executor` 变量保存它的引用。

如果只用如下代码：
```Java
ExecutorService executor =
    Executors.newSingleThreadExecutor();
```
使用 Java 提供的默认线程工厂创建工作线程，线程名称通常是`pool-1-thread-1`。

与上面的区别是，创建一个线程，名称为`ai-worker`。