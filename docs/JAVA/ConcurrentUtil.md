## 线程池:ExecutorService
### 个人理解
基于目前使用JDK8版本，java并无类似于其他语言的协程这种更为轻量的调度单位，异步的处理还是要落在线程身上，而为了减少创建和销毁的开销，线程池自然也就存在了。在工作中也不少接触线程池，主要用于异步处理任务
> tips:协程是基于用户态调度的，而线程创建、销毁、调度都是在内核态完成的，所以协程更为轻量
### 线程池基本参数
1. 核心线程数：直到线程池被回收才被回收的线程数
2. 最大线程数：当核心线程数满且队列满时会按照最大线程数来创建线程，最大线程数也满后执行拒绝策略，根据线程空闲时间(keepAliveTime)回收线程
3. 回收时间：keepAliveTime只能回收最大线程数，allowCoreThreadTimeOut可以回收核心线程
4. 队列：核心线程数满后新任务进入队列
5. 拒绝策略：队列满，最大线程数满后执行的拒绝策略
6. 
补充:业务中使用线程池通常需要设置线程池的名称前缀，方便后续排查问题，类似于这样
```java
    ExecutorService es = new ThreadPoolExecutor(3,
            6, 120L, TimeUnit.SECONDS,
            new ArrayBlockingQueue<Runnable>(200),
            new ThreadFactoryBuilder().setNameFormat("test-pool-%d").build(),
            new ThreadPoolExecutor.AbortPolicy());
```
通过```new ThreadFactoryBuilder().setNameFormat("test-pool-%d").build()```来控制线程池创建的线程前缀

### Future 带返回值的任务处理

通常异步处理是无需返回值的，在需要拿到线程执行结果的场景，我们就需要用到Future,Future提供了三种带返回值的Submit

```java
    //Callbale有回调，此时Future可以拿到结果
    <T> Future<T> submit(Callable<T> task);
    //Runnable接口，可以传入参数让主线程和线程池的子线程共享变量，参考下面代码1.1
    <T> Future<T> submit(Runnable task, T result);
    //Runnable接口无返回值，返回的Future仅可使用isDone()判断是否结束
    Future<?> submit(Runnable task);

    //1.1示例
    TestEntity entity = new TestEntity();
    entity.setStr("1");
    Future<TestEntity> future = es.submit(new MyRunnable(entity),entity);
    System.out.println(entity.getStr()); //1
    System.out.println(future.get().getStr()); //3

    class MyRunnable implements Runnable{
    private TestEntity entity;
    public MyRunnable(TestEntity entity) {
        this.entity = entity;
    }
    @Override
    public void run() {
        entity.setStr("3");
    }
```
我们看一下Future的定义：
```java
    //取消任务
    boolean cancel(boolean mayInterruptIfRunning);
    //是否已取消
    boolean isCancelled();
    //是否执行结束
    boolean isDone();
    //获取执行结果 , 如果get的时候线程还没有返回执行结果，则调用线程阻塞
    V get() throws InterruptedException, ExecutionException;
    //获取执行结果，设定超时时间，如果get的时候线程还没有返回执行结果，则调用线程直到阻塞至超时时间，抛出异常
    V get(long timeout, TimeUnit unit) 
    throws InterruptedException, ExecutionException, TimeoutException;

```

## 处理有聚合关系的任务:CompletableFuture
jdk8提供了异步编程的工具CompletableFuture，无需手工维护线程，语义清晰让代码更专注于业务，来看下类方法的定义，CompletableFuture不指定线程池的情况下默认使用forkjoinpool

```java
    //有返回值执行
    public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier);
    //有返回值指定线程池
    public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier,
                                                       Executor executor);
    //无返回值执行
    public static CompletableFuture<Void> runAsync(Runnable runnable);
    //无返回值指定线程池
    public static CompletableFuture<Void> runAsync(Runnable runnable,
                                                   Executor executor);
```

CompletableFuture处理串行关系:

```java
    CompletionStage<R> thenApply(fn);
    CompletionStage<R> thenApplyAsync(fn);
    CompletionStage<Void> thenAccept(consumer);
    CompletionStage<Void> thenAcceptAsync(consumer);
    CompletionStage<Void> thenRun(action);
    CompletionStage<Void> thenRunAsync(action);
    CompletionStage<R> thenCompose(fn);
    CompletionStage<R> thenComposeAsync(fn);
```

CompletableFuture处理AND关系:

```java
    CompletionStage<R> thenCombine(other, fn);
    CompletionStage<R> thenCombineAsync(other, fn);
    CompletionStage<Void> thenAcceptBoth(other, consumer);
    CompletionStage<Void> thenAcceptBothAsync(other, consumer);
    CompletionStage<Void> runAfterBoth(other, action);
    CompletionStage<Void> runAfterBothAsync(other, action);
```

CompletableFuture处理OR关系:

```java
    CompletionStage applyToEither(other, fn);
    CompletionStage applyToEitherAsync(other, fn);
    CompletionStage acceptEither(other, consumer);
    CompletionStage acceptEitherAsync(other, consumer);
    CompletionStage runAfterEither(other, action);
    CompletionStage runAfterEitherAsync(other, action);
```
异常处理:
```java
    CompletionStage exceptionally(fn);
    CompletionStage<R> whenComplete(consumer);
    CompletionStage<R> whenCompleteAsync(consumer);
    CompletionStage<R> handle(fn);
    CompletionStage<R> handleAsync(fn);
```
## 代码示例
### 串行
```java
        CompletableFuture<String> f0 =
                CompletableFuture.supplyAsync(() -> "Hello")
                        .thenApply(s -> s + " World")
                        .thenApply(String::toUpperCase);
        System.out.println(f0.join());
```
### AND
```java
        CompletableFuture<String> future1 =
                CompletableFuture.supplyAsync(() -> {
                    return "F2";
                }).thenCombine(CompletableFuture.supplyAsync(() -> {
                    return "F3";
                }), (f2, f3) -> {
                    return f2 + f3;
                });
        System.out.println(future1.join());
```
### OR
```java
        CompletableFuture<String> f1 =
                CompletableFuture.supplyAsync(() -> {
                    return "F1";
                });
        CompletableFuture<String> f2 =
                CompletableFuture.supplyAsync(() -> {
                    return "F2";
                });
        CompletableFuture<String> f3 =
                f1.applyToEither(f2, s -> s);
        System.out.println(f3.join());
```
###　异常处理
```java
        // /0后抛出异常
        CompletableFuture<Integer> future1 = 
            CompletableFuture.supplyAsync(() -> (1 / 0))
                    .thenApply(r -> r + 1)
                    .exceptionally(e -> 0);
        System.out.println(future1.join());

        //或者使用回调
        CompletableFuture<Integer> future1 = 
            CompletableFuture.supplyAsync(() -> (1 / 0))
            .whenComplete((r, e) -> {
                System.out.println("返回值:" + r);
                System.out.println("异常信息:" + e.toString());
            });
```
## 处理批量有序任务:CompletionService
### 个人理解
## Map/Reduce模型:Fork/Join
### 个人理解