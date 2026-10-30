# 复习Lambda 表达式

## 普通 Java 方法

```Java
public int add(int a, int b) {
    return a + b;
}
```

调用：

```Java
int result = add(10, 20);
```

这里相当于：

```Plain Text
a = 10
b = 20

return 30
```

## Lambda

> 在 Java 中，Lambda 表达式可以用来简洁地实现函数式接口的抽象方法。

看一个函数式接口：

```Java
@FunctionalInterface
public interface Calculator {

    int add(int a, int b);

}
```

这个接口只有一个抽象方法：

```Java
int add(int a, int b);
```

于是可以使用 Lambda 表达式实现这个接口。

### 不使用 Lambda

```Java
Calculator calculator = new Calculator() {

    @Override
    public int add(int a, int b) {
        return a + b;
    }

};
```

### 使用 Lambda

```Java
Calculator calculator = (a, b) -> a + b;
```

### Lambda 不需要声明参数类型

回到这个接口：

```Java
public interface Calculator {

    int add(int a, int b);

}
```

编译器看到：

```Java
Calculator calculator = (a, b) -> a + b;
```

就知道：

- `calculator` 的类型是 `Calculator`。
- `Calculator` 只有一个抽象方法 `add()`。
- `add()` 要求两个 `int` 参数。
- 因此，Lambda 中的 `a` 和 `b` 都是 `int`。

这叫做 类型推断（Type Inference）。

所以，以下两种写法完全等价：

```Java
Calculator calculator = (a, b) -> a + b;
```

```Java
Calculator calculator = (int a, int b) -> a + b;
```

可以不写类型，但不代表参数没有类型。

Java 编译器会根据上下文推断类型。

### Lambda 参数由谁传入？

由上例，我们已经定义好了：

```Java
int result = calculator.add(10, 20);
```

Lambda 才会接收到实际参数：

```Plain Text
a = 10
b = 20
```

最终返回：

```Plain
30
```

也就是说：

Lambda 左边声明形参，真正执行时，调用方负责传入实参。

这和普通方法的传参机制是一样的。

### Lambda箭头两端

基本结构：

```Java
(参数列表) -> 表达式或方法体
```

#### 箭头左端 —— 形参列表

Lambda 箭头 `->` 左边就是传入的形参（形式参数）。

例如：

1. 一个参数

```Java
x -> x * 2
```

- 左边 `x`：形参，接收传入的值。
    
- 右边 `x * 2`：计算并返回结果。
    

2. 多个参数

```Java
(a, b) -> a + b
```

- 左边 `a, b`：两个形参。
    
- 右边 `a + b`：计算并返回结果。
    

3. 没有参数

```Java
() -> {
    return 100;
}
```

- 左边 `()`：表示不接收任何参数。
    
- 右边：方法体，返回 `100`。
    

还有一个关键点：Lambda 左边的形参类型由目标函数式接口的抽象方法决定。

例如：

```Java
Callable<Integer> task = () -> 100;
```

因为 `Callable.call()` 没有参数，所以 Lambda 左边必须是 `()`。

而之前看到的[为什么这里是 task -> new Thread(task, "ai-worker")](<./为什么这里是 task -%3E new Thread(task, "ai-worker").md>)：

```Java
task -> new Thread(task, "ai-worker")
```

对应的是 `ThreadFactory.newThread(Runnable r)`，所以这里的 `task` 实际上是一个 `Runnable` 类型的形参。

> 这里有一个问题，`FutureGetLab`里没有做实参传入，但是这里需要传入一个`Runnable`对象，问题在于，这个方法由线程池负责调用，所以在代码里没有看到哪里做了实参传入，因为当线程池创建工作线程时，会自动传入一个`Runnable`实参。

#### 箭头右端 -- 表达式或完整方法体

形式一：单个表达式

```Java
Callable<Integer> task = () -> 100;
```

形式二：完整的方法体

`Callable`[这里Callable方法应该怎么写](<./什么是 ExecutorService.submit()?/这里Callable方法应该怎么写.md>)调用：

```Java
Callable<Integer> task = () -> {
    System.out.println("执行任务");
    return 100;
};
```

`Function`调用：

```Java
Function<Integer, Integer> function = x -> {
    return x * 2;
};
```

那么返回值是谁接收？谁调用这个方法，谁就接收返回值。

```Java
Integer a = task.call();
```

```Java
Integer result = function.apply(10);
```
