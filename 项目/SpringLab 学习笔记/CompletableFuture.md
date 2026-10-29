# Future——`FutureGetLab`

先声明一个线程

```Java
ExecutorService executor = Executors.newSingleThreadExecutor(
    task -> new Thread(task, "ai-worker")
);
```

>[[复习 Lambda 表达式.md|复习 Lambda 表达式]]
> 
>[[为什么这里是 task -> new Thread(task, "ai-worker").md|为什么这里是 task -> new Thread(task, "ai-worker")]]
