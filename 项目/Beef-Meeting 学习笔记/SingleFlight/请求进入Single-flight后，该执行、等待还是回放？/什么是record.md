# 什么是record

## record基础概念

`record`（记录类）是一种特殊的类，主要用于存储数据。

> record 是 Java 提供的一种简化类声明的方式。它特别适合用来创建只负责保存和传递数据的对象。

假设需要定义一个`User`类，保存用户的姓名和年龄。

使用普通`class`：

```Java
public final class User {

    private final String name;
    private final int age;

    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public int getAge() {
        return age;
    }
}
```

这里为了保存两个字段，我们需要编写：

```Plain Text
- 成员变量
    
- 构造方法
    
- Getter 方法
```

而且，如果想实现完整的值比较和字符串输出，还需要编写 `equals()`、`hashCode()` 和 `toString()`。

如果使用`record`：

```Java
public record User(String name, int age) {
}
```

只需要这一行代码，编译器会自动生成：

1. 两个 `private final` 字段。
    
2. 一个接收 `name` 和 `age` 的构造方法。
    
3. `name()` 和 `age()` 访问方法。
    
4. `equals()` 方法。
    
5. `hashCode()` 方法。
    
6. `toString()` 方法。

也就是说，record 可以帮我们省去大量重复代码。

## record怎么使用

```Java
public class Main {

    public static void main(String[] args) {

        User user = new User("张三", 20);

        System.out.println(user.name());

        System.out.println(user.age());

        System.out.println(user);
    }
}
```

输出：

```Plain Text
张三
20
User[name=张三, age=20]
```

`record`与普通 JavaBean 有个区别，JavaBean 通过:

```Java
user.getName();
```

`record`自动生成的访问方法是：

```Java
user.name();
```

不是`getName()`这种形式的 Getter。

## record 最重要的特性：不可重新赋值

record 的每个组件都会对应一个 `final` 字段。

```Java
public record User(String name, int age) {
}
```

创建对象后：

```Java
User user = new User("张三", 20);

user.name(); // 可以读取

user.name = "李四"; // 编译错误
```

因为 `name` 是 `private final` 的，不能重新赋值。

但是，需要特别注意：

record 并不是深度不可变的。

如果字段保存的是一个可变对象，比如 `List`，虽然不能让字段指向另一个 List，但仍然可能修改原 List 里的元素。

## record 也可以写自己的方法

```Java
public record User(String name, int age) {

    public boolean isAdult() {
        return age >= 18;
    }
}
```

调用：

```Java
User user = new User("张三", 20);

System.out.println(user.isAdult()); // true
```

也可以使用紧凑构造方法进行参数校验：

```Java
public record User(String name, int age) {

    public User {
        if (age < 0) {
            throw new IllegalArgumentException("年龄不能小于0");
        }
    }
}
```

这里的 `public User { ... }` 就是 record 特有的紧凑构造方法写法，编译器会自动处理组件字段的初始化。