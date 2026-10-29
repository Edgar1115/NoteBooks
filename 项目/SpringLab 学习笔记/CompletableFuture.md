# Future——`FutureGetLab`

先声明一个线程

```Java
ExecutorService executor = Executors.newSingleThreadExecutor(
    task -> new Thread(task, "ai-worker")
);
```

>[复习 Lambda 表达式](<%E5%A4%8D%E4%B9%A0 Lambda %E8%A1%A8%E8%BE%BE%E5%BC%8F.md>)
> 
>[为什么这里是 task -> new Thread(task, "ai-worker")](<%E4%B8%BA%E4%BB%80%E4%B9%88%E8%BF%99%E9%87%8C%E6%98%AF task -%3E new Thread(task, %22ai-worker%22).md>)
