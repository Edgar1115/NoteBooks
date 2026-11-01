# `Callable<T> task` 这里函数应该怎么写

> 复习 `Callable<T>` 函数式接口。

`Callable<T>`

```Java
@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception;
}
```

方法一：使用 Lambda 表达式

```Java
ExecutorService executor = Executors.newSingleThreadExecutor();

Callable<Integer> task = () -> {
    System.out.println("正在执行任务");
    return 100;
};

Future<Integer> future = executor.submit(task);

Integer result = future.get();
System.out.println(result);

executor.shutdown();
```

方法二：把任务写成一个独立的方法

定义一个普通 Java 方法：

```Java
public static String executeAiTask() throws InterruptedException {
    Thread.sleep(3000);
    return "AI回答";
}
```

再通过 `Callable` 提交：

```Java
ExecutorService executor = Executors.newSingleThreadExecutor();

// 可以简化为方法引用：Callable<String> task = MyClass::executeAiTask;
// 其中 MyClass 是定义这个静态方法的类名。
Callable<String> task = () -> executeAiTask();

Future<String> future = executor.submit(task);

System.out.println(future.get());

executor.shutdown();
```

方法三：创建一个真正实现 Callable 的类

如果不使用 Lambda，也可以直接实现接口：

```Java
public class AiTask implements Callable<String> {

    @Override
    public String call() throws Exception {
        Thread.sleep(3000);
        return "AI回答";
    }
}
```

然后：

```Java
Callable<String> task = new AiTask();

Future<String> future = executor.submit(task);
```

和第一种 Lambda 写法的作用相同。

---

## 单节 Q&A

Q1: Callable 不是要有抛出异常吗？这里没有抛出异常呢?

A1: `Callable` 的 `call()` 方法允许抛出异常，但不要求必须抛出异常。

Q2:  `Future<Integer> future = executor.submit(task);` 这里把 task 放进去了，是已经开始执行了，还是没有执行？不是要执行 .call()才算执行 Callable 吗？

A2: 调用 `executor.submit(task)` 后，任务就已经提交给线程池，线程池会自动调用 `task.call()`，不需要手动调用。什么时候执行由线程池和操作系统的线程调度决定。

Q3: 当我调用 future.get()，如果这个时候 submit 没有执行，会发生阻塞对吧？阻塞是future.get()后的代码都不执行了吗？ ^271855

A3: **如果调用 `future.get()` 的时候，对应的任务还没有执行完，那么当前调用 `get()` 的线程就会阻塞**。但不影响其他线程。
