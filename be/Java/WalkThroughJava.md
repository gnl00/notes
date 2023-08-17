# Walk Though Java

## 多线程

### 多线程异常处理方案

在使用 try-catch 的时候有一个条件通常会被忽略：try-catch 只能处理当前线程抛出的异常，在多线程情况下是无法 catch 到其他线程抛出的异常的。

**看个例子**

```java
public static void main(String[] args) {
    Test t = new Test();
    try {
        t.testMultiThreadException();
    } catch (StreamInputOutPutException e) {
        System.out.println("main thread catch exception: " + e.getMessage()); // catch failed
    }
}

public void testMultiThreadException() {
    new Thread(() -> {
        System.out.println("thread do something");
        throw new StreamInputOutPutException();
    }).start();
}
```

理想情况下，我们能在 main 线程中 catch 到其他线程抛出的 StreamInputOutPutException。但是运行结果如下：

```shell
thread do something
Exception in thread "Thread-0" com.demo.exception.StreamInputOutPutException: Input/output error
	at com.demo.test.Test.lambda$testMultiThreadException$0(Test.java:30)
	at java.lang.Thread.run(Thread.java:750)
```

可以看到，StreamInputOutPutException 并没有被 main 线程 catch 到。

**如何处理呢？**

1、我们能很简单的想到，在其他线程里面 try-catch 处理异常就可以。是的，这个办法行得通。将上面的代码稍作修改。

```java
public void testMultiThreadException() {
    new Thread(() -> {
        try {
            threadTask();
        } catch (StreamInputOutPutException e) {
            System.out.println("xx-thread can handle this exception: " + e.getMessage());
        }
    }).start();
}

public void threadTask() {
    System.out.println("thread do something");
    throw new StreamInputOutPutException();
}
```

```shell
thread do something
xx-thread can handle this exception: Input/output error
```

运行效果和我们期望的一样，成功 catch 到了该异常。

2、但是，如果存在很多个线程这样子操作就会多出很多代码。还有一个办法就是使用 Thread.UncatchExceptionHandler 来帮助我们在多线程中处理异常。

首先，新建自定义的 CustomThreadUncaughtExceptionHandler：

```java
public class CustomThreadUncaughtExceptionHandler implements Thread.UncaughtExceptionHandler {
    @Override
    public void uncaughtException(Thread t, Throwable e) {
        System.out.println("thread " + t.getName() + " catch exception: " + e.getMessage());
    }
}
```

然后稍微修改之前的代码：

```java
public void testMultiThreadException() {
    Thread th = new Thread(() -> {
        threadTask();
    });
    th.setUncaughtExceptionHandler(new CustomThreadUncaughtExceptionHandler());
    th.start();
}
```

输出结果如下：

```shell
thread do something
thread Thread-0 catch exception: Input/output error
```

也成功的捕获到了多线程情况下抛出的异常。

此外还可以使用 Thread 类的静态方法：

```java
public static UncaughtExceptionHandler getDefaultUncaughtExceptionHandler(){
    return defaultUncaughtExceptionHandler;
}
```

统一设置 UncaughtExceptionHandler。

3、线程组。

除了给单个线程设置 UncaughtExceptionHandler，还可以给一组线程设置 UncaughtExceptionHandler。

```java
class MyThread implement Runnable {
    // ...
}
ThreadGroup threadGroup = new ThreadGroup("线程组 A") {
    @Override
    public void uncaughtException(Thread t, Throwable e) {
        super.uncaughtException(t, e);
        log.error("线程组内捕获到线程[{},{}]异常", t.getId(), t.getName(), e);
    }
};
new Thread(threadGroup, new MyThread(), "线程 1").start();
Thread th2 = new Thread(threadGroup, new MyThread(), "线程 2");
th2.setUncaughtExceptionHandler(new CustomThreadUncaughtExceptionHandler()); // 优先使用为线程单独设置的 UncaughtExceptionHandler
```

4、线程池。

线程池也支持设置 UncaughtExceptionHandler，需要在实例化线程池的时候为其设置。

```java
ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
    2,
    4,
    1L,
    TimeUnit.MINUTES,
    new LinkedBlockingDeque<>(1024),
    new ThreadFactory() {
        @Override
        public Thread newThread(Runnable r) {
            Thread thread = new Thread(r);
            // setUncaughtExceptionHandler
            thread.setUncaughtExceptionHandler(new CustomThreadUncaughtExceptionHandler());
            return thread;
        }
    }
);
```

但是 UncaughtExceptionHandler 只能捕获到由 execute 方法提交的任务抛出的异常，对于 submit 方法提交的任务抛出的异常是无法捕获的。

此时就需要使用 afterExecute 方法来捕获 submit 提交的任务的异常。

```java
ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
    2,
    4,
    1L,
    TimeUnit.MINUTES,
    new LinkedBlockingDeque<>(1024),
    r -> {
        Thread thread = new Thread(r);
        // setUncaughtExceptionHandler
        thread.setUncaughtExceptionHandler(new CustomThreadUncaughtExceptionHandler());
        return thread;
    }
) {
    @Override
    protected void afterExecute(Runnable r, Throwable t) {
        super.afterExecute(r, t);
        if (r instanceof FutureTask<?>) {
            try {
                ((FutureTask<?>)r).get(); // 此时得到线程池内的任务抛出的异常
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            } catch (ExecutionException e) {
                throw new RuntimeException(e);
            }
        }
    }
};
```

<br>



## 参考

**多线程**

* https://developer.aliyun.com/article/949454