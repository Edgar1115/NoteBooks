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
