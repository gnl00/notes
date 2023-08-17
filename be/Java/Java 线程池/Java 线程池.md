# Java 线程池

> 线程池是 Java 中用于管理和复用线程的机制。它包含了一个线程队列，可以用来创建并发执行的任务，提高程序的执行效率

<br>

## 前言

**线程池特点**

1、线程复用，降低资源消耗，通过复用已创建的线程降低线程创建销毁引起的消耗

2、管理线程，线程是稀缺资源，如果无限创建，不仅会消耗系统资源，还会降低系统稳定性，使用线程池可以进行统一的分配

3、提高响应速度，当任务到达时线程就可执行，减少了线程创建的时间

<br>

## 接口与类

### Executor

> Executor 是线程池操作最顶层的类，用于运行提交的线程任务。此接口将线程任务的提交和运行机制进行解耦，包括线程任务的使用和调度等细节
>
> 通常使用 Executor 而不是显式创建线程，相比于遍历线程集合运行每一个任务，更常用的方法如下：
>
> ```java
> Executor executor = anExecutor;
> executor.execute(new RunnableTask1());
> executor.execute(new RunnableTask2());
> ```
>
> <br>
> Executor 接口并不严格要求执行必须是异步的，Executor 可以立即在调用者的线程中运行提交的任务：
>
> ```java
> class DirectExecutor implements Executor {
> 	public void execute(Runnable r) {
>     		r.run();
>   	}
> }
> ```
>
> <br>
> 通常情况下，任务是在调用者线程之外的某个线程中执行的：
>
> ```java
> class ThreadPerTaskExecutor implements Executor {
>       public void execute(Runnable r) {
>        	new Thread(r).start();
>       }
> }
> ```

<br>

> **内置方法**
>
> - execute，运行线程任务，无返回值
>

<br>

> **子类与实现类**
>
> - ExecutorService
>
>   Java 提供的线程池都实现了 ExecutorService 接口，相对 Executor 它的扩展性更强
>
> - ThreadPoolExecutor
>
>   提供了一个可扩展的线程池实现
>
> - Executors
>
>   提供方便线程池创建的工厂方法

<br>

> **内存一致性原则**
>
> 将 Runnable 对象（线程任务）提交给 Executor 的操作 happens-before 线程任务执行开始

<br>

### ExecutorService

> 提供线程池的提交/管理/运行/中止方法，还可以创建用于跟踪一个或多个异步任务进度的 [Future](#Future)

```java
public interface ExecutorService extends Executor
```

<br>

> **内置方法**
>
> - submit
>
>   可以看成 Executor.execute(Runnable) 方法的一个扩展，返回的 Future 可用于线程任务取消/等待完成。
>
> - invokeAny/invokeAll
>
>   批量执行线程任务，等待至少一个/全部任务完成
>
> - shutdown
>
>   拒绝新任务，允许任务队列中的所有任务都执行完毕再关闭线程池
>
> - shutdownNow
>
>   拒绝新任务，直接关闭线程池

<br>

> **内存一致性原则**
>
> 线程任务的提交 happens-before 线程任务的执行；线程任务执行完成 happens-before Future#get

<br>

### AbstractExecutorService

> 使用 [RunnableFuture](#RunnableFuture) 实现了 submit/invokeAny/invokeAll 方法

```java
public abstract class AbstractExecutorService implements ExecutorService
```

<br>

### ThreadPoolExecutor

> Thread pools address two different problems
>
> - 线程池在执行大量的异步任务时，可以减少每个任务执行时的开销，拥有更高的性能
> - 线程池可以更好的管理和约束资源，包括线程在执行一系列任务时的开销
>
> <br>
>
> ThreadPoolExecutor 是 Java 线程池的核心实现类，维护基础的线程池信息，如线程池的核心线程数/最大工作线程数/非核心线程空闲存活时间/线程工厂/线程工作队列/队列拒绝策略等。
>
> <br>
>
> However, programmers are urged to use the more convenient Executors factory methods 
>
> * Executors.newCachedThreadPool (unbounded thread pool, with automatic thread reclamation)
> * Executors.newFixedThreadPool (fixed size thread pool) 
> * Executors.newSingleThreadExecutor (single background thread)
> * Executors.newSingleThreadScheduledExecutor (run after a given delay, or to execute periodically)
>
> <br>
>
> Otherwise, use the following guide when manually configuring and tuning this class
>
> * Core and maximum pool sizes
> * On-demand construction
> * Creating new threads
> * Keep-alive times
> * Queuing
> * Queue maintenance
> * Rejected tasks
> * Hook methods
> * Finalization



> **Core and maximum pool sizes**
>
> A ThreadPoolExecutor will automatically adjust the pool size according to the bounds set by corePoolSize and maximumPoolSize.
>
> * 新任务提交时，如果当前存活的线程数小于 corePoolSize，即使存活的其他线程是空闲的，也会创建新线程处理新提交的任务
>
> * 新任务提交时，如果当前存活线程数大于 corePoolSize 但是小于 maximumPoolSize，只有线程等待队列满了的时候才会创建新线程
>
> <br>
>
> 如果 corePoolSize 与 maximumPoolSize 相等，则相当于 fixed-size 的线程池；如果将 maximumPoolSize 设置成无限大，如 Integer.MAX_VALUE，就表示线程池允许同时运行任意数量的线程。
>
> <br>
>
> 通常来说，corePoolSize 和 maximumPoolSize 只会在构建线程池时指定，但也可以通过 setCorePoolSize 和 setMaximumPoolSize 方法进行动态更改
>
> <br>
>
> 实际上在创建线程时，并没有给线程做标记，因此无法区分核心线程与非核心线程，主要根据当前线程的数量来处理。只要当前的运行线程数小于 corePoolSize，就认为存活的线程都是核心线程。线程池根据当前线程数量判断是否退出线程，而不是根据是否是核心线程。



> **On-demand construction**
>
> 执行线程按需创建，默认情况下，即使是核心线程，只有当任务提交的时候才会创建。但是可以使用 prestartCoreThread 或 prestartAllCoreThreads 方法来预先启动核心线程



> **Creating new threads**
>
> * 新线程会使用 ThreadFactory 线程工厂来创建，如果没有特别说明，则使用 Executors.defaultThreadFactory
> * 默认线程工厂创建的线程都在同一个 ThreadGroup 中，有着同样的 NORM_PRIORITY 优先级，并且都是非守护线程
> * 可以通过使用自定义的 ThreadFactory 修改新创建的线程的基本信息，如线程名、线程组、线程优先级、是否是守护线程等
> * 如果 ThreadFactory 创建线程时返回 null，线程创建失败。线程池会继续执行，但是可能不能执行任何任务
> * 线程应拥有“修改线程”运行时权限，如果工作线程或其他使用该池的线程不具备此权限，服务可能会降低。配置变化可能不会及时生效，而且关闭的线程池可能保持在一个可能终止但没有完成的状态中



> **Keep-alive times**
>
> * 如果当前池中的线程数大于 corePoolSize 多余的线程在空闲时间超过 keepAliveTime 后将被终止
>
> * 默认情况下，只有当线程数大于 corePoolSize 时才会应用 keep-alive 策略。但可以使用 allowCoreThreadTimeOut(boolean) 方法将超时策略应用到核心线程上，前提是 keepAliveTime 值不为零



> **Queuing**
>
> 任何 BlockingQueue 都可以用来维护提交的任务
>
> * 如果运行的线程少于 corePoolSize，线程池总是倾向于创建新的线程而不是入队
> * 如果运行的线程大于等于 corePoolSize，线程池总是倾向于入队，而不是创建新线程
> * 如果一个任务请求无法入队，则会创建新的线程来处理，除非线程数即将超过 maximumPoolSize。此时将触发拒绝策略
>
> <br>
>
> There are three general strategies for queuing:
>
> * Direct handoffs
>
>   同步队列 SynchronousQueue，将任务交给线程，不持有任何任务。如果没有线程可以立即运行任务，尝试入队的任务将会入队失败，此时将会创建新的线程来处理。通常需要无界的 maximumPoolSizes 才能避免对新提交的任务的拒绝，但这会导致无限的线程增长
>
> * Unbounded queues
>
>   无界队列，比如 LinkedBlockingQueue，新任务在所有 corePoolSize 线程都忙时直接入队，创建的线程数永远不会超过 corePoolSize，此时 maximumPoolSize 将无效。
>
>   当每个任务都是独立的时候，这种队列可能是合适的。但是如果两个任务之间有关联，一个在队列头，一个在队列尾，将会导致头任务阻塞很长一段时间等待尾任务完成。
>
>   Java 内置的线程池默认使用 LinkedBlockingQueue 队列来维护任务。
>
> * Bounded queues
>
>   有界队列，如 ArrayBlockingQueue，如果存在大量任务，可能会经常阻塞。但是可以防止资源耗尽
>
> <br>
>
> 线程池可容纳的最大任务数 `maxTaskSize = maximumPoolSize + workQueue.size()`，超过能容纳的最大任务树就会触发拒绝策略，抛出 RejectedExecutionException



> **Rejected tasks**
>
> * 线程池关闭后，新提交的任务将会被拒绝
> * 如果线程池使用的是有界队列，正在运行的线程数大于 maximumPoolSize，并且队列已满，会触发拒绝策略
> * 如果任务在执行中主动调用 RejectedExecutionHandler.rejectedExecution 方法，会拒绝执行任务，抛出未受检查的异常 RejectedExecutionException
>
> <br>
>
> **拒绝策略**
>
> * ThreadPoolExecutor.AbortPolicy
>
>   默认拒绝策略，在拒绝执行时抛出 RejectedExecutionException 异常，线程池正常运行
>
> * ThreadPoolExecutor.CallerRunsPolicy
>
>   调用 execute 方法的线程自己执行任务，提供了一个简单的反馈控制机制，但是会减缓新任务的提交速度
>
> * ThreadPoolExecutor.DiscardPolicy
>
>   不能执行的任务会被直接放弃
>
> * ThreadPoolExecutor.DiscardOldestPolicy
>
>   丢弃队列头的任务，再将当前任务入队
>
> <br>
>
> 可以实现 RejectedExecutionHandler 接口自定义和使用其他类型的 RejectedExecutionHandler



> **Hook methods**
>
> ThreadPoolExecutor 还提供任务执行前后的勾子方法
>
> * beforeExecute(Thread, Runnable)
> * afterExecute(Runnable, Throwable)
> * terminated()
>
> <br>
>
> 勾子函数可用作线程执行环境的检查/初始化 ThreadLocal/日志操作等。线程 terminated 方法也可以被重写，用作中止前后进行清理操作等。如果勾子函数或回调方法抛出异常，内部工作线程可能反过来失败并突然终止



> **Queue maintenance**
>
> * 可以通过 getQueue 方法获取到工作队列，进行监控或者 debug 等操作，不鼓励进行除此之外的其他操作
>
> * remove(Runnable) 和 purge 方法能够协助进行工作队列内存回收



> **Finalization**
>
> * 不被引用并且没有剩余线程的池将被自动关闭
> * 如果想要确保未被引用的线程池不调用 shutdown 方法也会被回收，必须通过设置适当的 keep-alive 时间、使用零个核心线程的下限，或者设置 allowCoreThreadTimeOut(boolean) 来使得未使用的线程最终被回收

```java
public class ThreadPoolExecutor extends AbstractExecutorService
```

<br>

#### 线程池状态

* RUNNING 运行状态，接受新任务，持续处理任务队列里的任务

* SHUTDOWN 调用 shutdown 方法会进入 SHUTDOWN 状态，不再接受新任务，但要处理任务队列里的任务。shutdown 方法会等待线程都执行完毕之后再关闭

* STOP 调用 shutdownNow 方法，不再接受新任务，不再处理任务队列里的任务，中断正在进行中的任务。shutdownNow 方法相当于调用每个线程的 interrupt 方法

* TIDYING 表示线程池正在进行清理工作，中止所有任务，销毁所有工作线程

* TERMINATED 表示线程池已停止运作，所有工作线程已被销毁，所有任务已被清空或执行完毕

<br>

#### 线程池种类

> Executors 类中预定义了几种常用的线程池
>
> * 预定义的线程池都是 ThreadPoolExecutor 的实例，只是在构造函数中传入的参数不同
>
> * 线程池执行 execute() 提交不需要返回值的任务，无法判断是否执行成功；线程池执行 submit() 提交有返回值的任务。返回 Future 对象，可以判断任务是否执行成功。

* FixedThreadPool

  线程数量固定的线程池，可控制线程最大并发数，超出的线程会在队列中等待

  ```java
  public static ExecutorService newFixedThreadPool(int nThreads) {
      return new ThreadPoolExecutor(nThreads, nThreads,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>());
  }
  ```

  

* CacheThreadPool

  可缓存/扩展的线程池，可灵活回收空闲线程，若无可回收，则新建线程

  ```java
  public static ExecutorService newCachedThreadPool() {
      return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                    60L, TimeUnit.SECONDS,
                                    new SynchronousQueue<Runnable>());
  }
  ```

  

* ScheduledThreadPool

  支持定时及周期性任务执行的线程池

  ```java
  public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
      return new ScheduledThreadPoolExecutor(corePoolSize);
  }
  
  public ScheduledThreadPoolExecutor(int corePoolSize) {
      super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
            new DelayedWorkQueue());
  }
  ```

  

* SingleThreadPool

  单一线程化的线程池，用唯一的线程来执行任务，保证所有任务按照顺序执行

  ```java
  public static ExecutorService newSingleThreadExecutor() {
      return new FinalizableDelegatedExecutorService
          (new ThreadPoolExecutor(1, 1,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>()));
  }
  ```

  

<br>

#### 自定义线程池

> 不推荐使用 Executors 定义的线程池，推荐通过 ThreadPoolExecutor 来创建自定义线程池
>
> * FixedThreadPool 和 SingleThreadPool 工作队列最大长度为 Integer.MAX_VALUE，可能会堆积大量任务请求，导致 OOM
> * CacheThreadPool 和 ScheduleThreadPool 允许创建的最大线程数量为 Integer.MAX_VALUE，可能会创建大量的线程，导致 OOM

```java
// 一般根据 CPU 核心数设置 corePoolSize = Runtime.getRuntime().availableProcessors() * 2
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.acc = System.getSecurityManager() == null ?
            null :
            AccessController.getContext();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

<br>

#### 线程池工作流程

> 1、线程池被创建之后，开始等待任务请求
>
> 2、当调用 execute 方法添加请求时
>
> * 正在运行的线程数小于 corePoolSize，立即创建线程执行任务
>
> * 正在运行的线程数大于等于 corePoolSize，任务入队等待
>
> * 队列已满，且正在运行的线程数量小于 maximumPoolSize，立刻创建线程执行新请求的任务
>
> * 正在运行的线程数大于 maximumPoolSize 且阻塞队列已满，触发拒绝策略
>
> 3、当一个线程完成任务，它就会从队列中取出下一个任务执行
>
> 4、当一个线程空闲时间超过 keepAliveTime，线程就会判断：当前运行的线程数是否大于 corePoolSize，是的话这个线程就会被关闭
>
> 5、线程池的所有任务完成后，线程数会保持在 corePoolSize

![任务调度流程](./assets/31bad766983e212431077ca8da92762050214.png)

<br>

### Future

> Future 表示异步任务的结果，任务完成后，可使用 get 方法获取；如果未完成，get 方法将会阻塞。还额外提供有检查任务是否完成和是否取消的方法。任务的取消可以使用 cancel 方法，如果任务已完成就不可被取消。
>
> 如果使用 Future 仅仅是为了获得可取消的任务，不需要返回值，可以使用 `Future<?>` 在任务执行完后，返回 null 作为结果。

```java
public interface Future<V>
```

> **内置方法**
>
> - get
> - cancel
> - isDone
> - isCancelled

**Usage**

```java
interface ArchiveSearcher {
    String search(String target);
}

class App {
    ExecutorService executor = ...
    ArchiveSearcher searcher = ...

    void showSearch(final String target) throws InterruptedException {
        Future<String> future = executor.submit(new Callable<String>() {
            public String call() {
                return searcher.search(target);
            }
        });
        displayOtherThings(); // do other things while searching
        try {
            displayText(future.get()); // use future
        } catch (ExecutionException ex) {
            cleanup();
            return;
        }
    }
}
```

<br>

### RunnableFuture

> A Future that is Runnable. 可执行的 Future，run 方法执行成功后可以通过 Future 自身的 get 方法获取到返回值。

```java
public interface RunnableFuture<V> extends Runnable, Future<V>
```

<br>

### FutureTask

> 一个可取消的异步操作。这个类提供了 Future 的一个简单实现，实现了：开始/取消/获取结果/检查是否完成方。只有操作完成之后才能获取到返回结果，否则会一直阻塞 get 方法的执行；操作结束之后不能被 restart 或者 cancelled（存在例外：runAndReset 方法）。
>
> 可以将 Callable 或者 Runnable 对象传递到 FutureTask 的构造函数，来创建 一个 FutureTask。

```java
public class FutureTask<V> implements RunnableFuture<V>
```

<br>

### CompletableFuture

> 一个明确会在将来完成的 Future。实现了 CompletionStage 接口，用于在异步任务进行时针对不同阶段进行不同操作。



> **内部方法**
>
> * complete
> * completeExceptionally
> * cancel
>
> 当存在多个线程同时执行以上三个方法时，只有一个能执行成功。



> **Overview**
>
> CompletableFuture 中可能会存在依赖于结果完成的动作，CompletableFuture 通过对结果字段的 CAS 操作来自动完成这一系列动作。
>
> 
>
> 返回的结果字段非空，说明 CompletableFuture 操作完成。CompletableFuture 使用一个 AltResult 来包装 null 值结果，同时也是使用 AltResult 来保存异常。



<br>

## ForkJoin 框架

### ForkJoinPool

> 本质上是一个 ExecutorService，是一个专门用来运行 ForkJoinTask 的线程池。同样提供了对于 Non-ForkJoinTask 的线程池运行入口、管理和监控的操作。
>
> 与其他 ExecutorService 不同的是，ForkJoinPool 支持一种叫做 ”work-stealing“ 的工作模式：线程池中的所有线程都会尝试去找到并执行<u>所有被提交到线程池中的任务（或者是被其他活跃任务创建的任务）</u>。这种工作模式在会有很多子任务产生的情况下，或者有很多小人物会被从外部客户端提交到线程池中运行时是很高效的。
>
> ForkJoinPool 提供一个静态的 commonPool() 方法，适用于绝大多数应用，可以被任何一个 ForkJoinTask 使用。使用 commonPool 还能够减少资源的使用（线程不活跃的时候会被慢慢回收，使用的时候自动恢复）。

> **三种运行方式**
>
> |                                | Call from non-fork/join clients | Call from within fork/join computations       |
> | ------------------------------ | ------------------------------- | --------------------------------------------- |
> | Arrange async execution        | execute(ForkJoinTask)           | ForkJoinTask.fork                             |
> | Await and obtain result        | invoke(ForkJoinTask)            | ForkJoinTask.invoke                           |
> | Arrange exec and obtain Future | submit(ForkJoinTask)            | ForkJoinTask.fork (ForkJoinTasks are Futures) |

```java
public class ForkJoinPool extends AbstractExecutorService
```

<br>

内部类 *WorkQueue* 维护一个 *work-stealing queues*，是 *Deques* 的一个变种，只支持 3 个操作：push，pop 和 poll（也叫做 steal）。push 和 pop 只能由线程本身调用，一个线程的 poll 操作可能会被其他外部线程调用。

*work-stealing queue* 内部的用到了 CAS 来获取下一个 task 进行执行。

> 关于 *work-stealing queue* 可以看看 [Dynamic Circular Work-Stealing Deque](http://research.sun.com/scalable/pubs/index.html) 和 [Idempotent work stealing](http://research.sun.com/scalable/pubs/index.html)。



<br>

### ForkJoinTask

> 运行在 ForkJoinPool 中的的任务的抽象任务类，ForkJoinTask 是一种轻量级的 Future。
>
> ForkJoinTask 线程比普通线程更加轻量，在可用资源受到限制的时候，在 ForkJoinPool 中使用少量的线程即可完成大量的 ForkJoinTask 任务。

```java
public abstract class ForkJoinTask<V> implements Future<V>, Serializable
```

<br>

ForkJoinTask 是一个抽象类，所以需要继承它的一系列子类来完成需要的功能。下面测试一下使用和不使用 ForJoinTask 的区别：

1、不使用 ForJoinTask。

```java
public void noThread() {
    long start = System.currentTimeMillis();
    long sum = 0;
    for (long i = 1; i <= 1000000000; i++) {
        sum += i;
    }
    System.out.println(sum);
    long end = System.currentTimeMillis();
    System.out.println("total time spent ===> " + (end - start));
}
```

```shell
executing ===> noThread
500000000500000000
total time spent ===> 352
```

2、使用 ForJoinTask。

```java
public class MyForkJoinTask extends RecursiveTask<Long> {
    private long from;
    private long to;
    private long maxRange;

    public MyForkJoinTask(long from, long to, long maxRange) {
        this.from = from;
        this.to = to;
        this.maxRange = maxRange;
    }

    @Override
    protected Long compute() {

        long range = to - from;
        long sum = 0;
        if (range >= maxRange) {
            long mid = (from + to) / 2;
            MyForkJoinTask left = new MyForkJoinTask(from, mid, maxRange);
            MyForkJoinTask right = new MyForkJoinTask(mid + 1, to, maxRange);

            // fork() asynchronously, execute this task in the pool the current task is running in
            left.fork();
            right.fork();

            // join() Returns the result of the computation when it is done
            sum += left.join();
            sum += right.join();
        } else {
            for (; from <= to ; from++) {
                sum += from;
            }
        }
        return sum;
    }
}
```

```java
public class MyForkJoinPool {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ForkJoinPool fjp = ForkJoinPool.commonPool();

        long start = System.currentTimeMillis();

        MyForkJoinTask task = new MyForkJoinTask(1, 1000000000, 100000000);
        ForkJoinTask<Long> resp = fjp.submit(task);
        System.out.println(resp.get());

        long end = System.currentTimeMillis();
        System.out.println("cost time ==> " + (end - start));
    }
}
```

```shell
500000000500000000
cost time ==> 90
```

计算所需要的时间只需要原先的四分之一。

> 在测试中发现，使用多线程来计算时间花费得更少：
>
> ```java
> public void withThread() {
>     AtomicLong globalSum = new AtomicLong();
> 
>     long from = 0;
>     long max = 1000000000;
>     long span = max/10;
> 
>     // 10 threads
>     for (int i = 1; i <= 10; i++) {
>         long finalFrom = from; // 0          1000000001
>         long to = span * i; // 1000000000 2000000000
>         new Thread(() -> {
>             long sum = 0;
>             for (long j = finalFrom; j <= to; j++) {
>                 sum += j;
>             }
>             System.out.println("sum now: " + sum);
>             long gs = globalSum.addAndGet(sum);
>             System.out.println("global sum now: " + gs);
>         }).start();
>         from = to + 1;
>     }
> }
> ```
>
> ```shell
> executing ===> withThread
> global sum now: 500000000500000000
> total time spent ===> 42
> ```
>
> 甚至只需要原先的九分之一的时间就可以完成计算。



<br>

### ForkJoinWorkerThread

> 由 ForkJoinPool 管理的线程，用来处理 ForkJoinTask 任务。
```java
public class ForkJoinWorkerThread extends Thread
```

在 ForkJoinPool 中有说明：

```java
/*
 * Creating workers. To create a worker, we pre-increment total
 * count (serving as a reservation), and attempt to construct a
 * ForkJoinWorkerThread via its factory. Upon construction, the
 * new thread invokes registerWorker, where it constructs a
 * WorkQueue and is assigned an index in the workQueues array
 * (expanding the array if necessary). The thread is then
 * started. 
 * Upon any exception across these steps, or null return
 * from factory, deregisterWorker adjusts counts and records
 * accordingly.  If a null return, the pool continues running with
 * fewer than the target number workers. If exceptional, the
 * exception is propagated, generally to some external caller.
 * Worker index assignment avoids the bias in scanning that would
 * occur if entries were sequentially packed starting at the front
 * of the workQueues array. We treat the array as a simple
 * power-of-two hash table, expanding as needed. The seedIndex
 * increment ensures no collisions until a resize is needed or a
 * worker is deregistered and replaced, and thereafter keeps
 * probability of collision low. We cannot use
 * ThreadLocalRandom.getProbe() for similar purposes here because
 * the thread has not started yet, but do so for creating
 * submission queues for existing external threads.
 */
```

> 在创建 worker 时，ForkJoinPool 会提前新增线程总数，并尝试通过 ForkJoinWorkerThread 工厂来创建 ForkJoinWorkerThread。
>
> 在创建时，新线程通过调用 ForkJoinPool#registerWorker 创建一个 WorkQueue，并将 WorkQueue 添加到 WorkQueue 数组中，表示线程启动完毕。
> 如果在这过程中出现了异常或者 ForkJoinWorkerThread 工厂返回了 null 值：
>
> * 调用 ForkJoinPool#deregisterWorker 调整线程数量，并进行相应的记录；
> * 如果返回了 null 值，ForkJoinPool 会继续运行；
> * 如果出现异常，通常会被抛出给外部调用者处理。



<br>

## 文章

* https://mp.weixin.qq.com/s/7jU0ci2tn5TwCvOyxdX6dA
