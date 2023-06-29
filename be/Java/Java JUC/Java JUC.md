# JUC

> JUC 指的是 `java.util.concurrent` 包及其子包下用于并发场景的类

## 基础框架

<br>

### AbstractOwnableSynchronizer

> 一个可以让线程独占的同步器，为锁和相关同步器的创建提供了一个基础的概念

```java
public abstract class AbstractOwnableSynchronizer
    implements java.io.Serializable
```

**内部属性/方法**

```java
// The current owner of exclusive mode synchronization.
private transient Thread exclusiveOwnerThread;

/**
 * A null argument indicates that no thread owns access. This method does not otherwise
 * impose any synchronization or volatile field accesses.
 */
protected final void setExclusiveOwnerThread(Thread thread) {
    exclusiveOwnerThread = thread; // 设置 null 值表示当前没有线程有访问权
}
```



<br>

### AbstractQueuedSynchronizer

> 提供一个基本框架，用于实现阻塞锁（依靠 FIFO 等待队列）和相关同步器（信号量、栅栏等）。此类是大多数依靠单一原子 int 值来表示状态的同步器的基础，它定义了一些实现同步机制需要使用的方法，提供给具体的锁和相关的同步器使用。

> **子类与实现类**
>
> * 子类必须定义用来修改同步机制状态 state 的 protected 方法，如同步锁资源被获取和释放时修改 state 的 acquire 和 release 方法。同时也要定义获取不到锁资源时用于排队的方法和实现阻塞机制的方法
> * 子类还可以维护除了 state 之外的其他变量，但是只有使用 getState/setState 和 compareAndSetState 方法更新 state 值才会被当作同步机制的实现逻辑，进行同步状态跟踪
> * 子类应该定义为非公开的内部辅助类，用于实现其封闭父类类的同步属性



> **锁的两种模式**
>
> AbstractQueuedSynchronizer 同时支持独占和共享模式，在不同模式下排队的线程使用的都是同一个 FIFO 队列，可以使用 isHeldExclusively 方法来检查当前线程是否独占有锁资源。
>
> 通常来说子类可以只实现两种模式中的一种，这种情况下，未实现的模式的方法不需要实现；子类也可以同时支持两种模式，如 ReadWriteLock。



> **Condition**
>
> ConditionObject 是 AbstractQueuedSynchronizer 的内部类，ConditionObject 实现了 Condition 接口，可以作为独占模式下锁资源获取与释放的条件来使用。
>
> 可以使用 getState 获取到的 state 作为参数，根据 Condition 条件调用 release/acquire 方法释放或者重入锁资源



> **序列化与反序列化**
>
> 对 AbstractQueuedSynchronizer 进行序列化操作会把 state 设置为 0，即无锁模式，线程等待队列不会进行序列化。需要自定义 readObject 方法，反序列化时将 state 和 等待队列恢复



> **自定义同步器**
>
> 如果想要实现自定义同步机制，需使用 getState/setState/compareAndSetState 方法来重新定义以下几个方法的逻辑
>
> * tryAcquire
> * tryRelease
> * tryAcquireShared
> * tryReleaseShared
> * isHeldExclusively
>
> 这几个方法是 AbstractQueuedSynchronizer 中的模版方法，默认抛出 UnsupportedOperationException 异常。
>
> 自定义实现时要求必须是内部线程安全的，并且调用流程应该是短暂，不阻塞的。重写这几个方法是使用 AbstractQueuedSynchronizer 的唯一方法，因为除此之外的其他方法基本上都是 final 声明的，不可被重写。



> **公平与非公平**
>
> * 虽然 AbstractQueuedSynchronizer 同步机制的实现依赖于内部的 FIFO 队列，但它并不强制使用 FIFO 来获取锁资源
> * 子类的实现为公平锁时，需要严格按照 FIFO 队列的要求来实现同步机制；子类的实现为非公平锁时，不需要按照 FIFO 要求实现同步机制。因为在获取锁资源时会先进行检查，所以非公平模式下，新获取的线程可能会在其他被阻塞和排队的线程之前获取到锁
> * 默认情况下（非公平模式），允许线程之间进行锁资源的争夺，此时吞吐量和可扩展性比较高。因为没获取到锁的线程会先进行自旋，并在自旋中尝试获取锁，自旋获取失败才会进入阻塞队列等待



#### 内部类

##### Node

> 等待队列的 Node 结点，等待队列是 CLH（Craig, Landin, and Hagersten） 锁队列的一个变种。CLH 队列通常用于实现自旋锁，在 AQS 中使用 CLH 队列来实现阻塞同步器，将线程的控制信息保存在其前驱结点中。
>
> 
>
> 每个线程中都有一个 status 字段，用来记录该线程是否应该阻塞（仅用作记录，不用做控制锁状态）。当前驱结点释放锁资源时，等待队列中阻塞的结点就会收到信号。等待队列中的第一个线程不能保证一定会获取到锁资源，只能保证它具有竞争锁的权利，如果获取失败会继续等待。

> ```
>      +------+  prev +-----+       +-----+
> head |      | <---- |     | <---- |     |  tail
>      +------+       +-----+       +-----+
> ```
>
> * 线程入队阻塞队列仅需要使用原子操作将线程结点插入等待队列的尾部，线程出队只需要将获取到锁资源的线程设置为头结点即可。耗时的操作是确定后继结点，可能会出现超时或者中断的情况
>
> * CHL 队列中的 prev 指针主要用作取消操作，如果一个线程结点被取消了，它的后继会重新指向前驱结点没被取消的前驱
>
> * 除了 prev 还有一个 next 指针指向后继结点，获取到资源的线程释放时，会通过 next 通知后继的结点进行争夺

```java
static final class Node
```



**内部属性**

```java
// Marker to indicate a node is waiting in shared mode
static final Node SHARED = new Node();
// Marker to indicate a node is waiting in exclusive mode
static final Node EXCLUSIVE = null;

/**
 * statue 字段，有以下几种取值 
 * SIGNAL
 * CANCELLED
 * CONDITION
 * PROPAGATE
 *
 * 普通同步机制下默认值是 0，如果使用 condition 则默认值是 CONDITION = -2
 */
volatile int waitStatus; // 

// waitStatus 属性的取值有以下情况 waitStatus value to indicate that
static final int CANCELLED =  1; // thread has cancelled due to timeout or interrupt, a thread with cancelled node never again blocks
static final int SIGNAL    = -1; // successor's thread needs unparking
static final int CONDITION = -2; // thread is waiting on condition
static final int PROPAGATE = -3; // the next acquireShared should unconditionally propagate
```

<br>

##### ConditionObject

> Condition implementation for a AbstractQueuedSynchronizer serving as the basis of a Lock implementation.

```java
public class ConditionObject implements Condition, java.io.Serializable
```

**ConditionObject 内部方法**

```java
/**
 * 实现条件中断等待
 * 1、阻塞当前线程，如果当前线程已中断，抛出 InterruptedException
 * 2、保存 getState 获取到的 state
 * 3、将保存的 state 传入 release 方法并执行
 * 4、阻塞直到被唤醒或者中断
 * 5、使用保存的调用指定的 acquire 方法重新获取锁资源
 * 6、如果在第四步时已中断，抛出 InterruptedException
 */
public final void await() throws InterruptedException {}

// 将等待时间最长的线程（如果存在的话）从这个条件的等待队列移到拥有锁的等待队列中
public final void signal() {}

// 将所有线程从这个条件的等待队列移到拥有锁的等待队列中
public final void signalAll() {}
```



<br>

#### 内部属性

```java
private volatile int state; // 同步机制的状态

/**
 * 等待队列的头结点，懒加载。除了初始化方法，只有 setHead 方法可以修改头结点。
 * 如果头结点存在，waitStatus 保证不会被设置为 CANCELLED
 */
private transient volatile Node head;

// 等待队列的尾结点，懒加载
// 仅可以通过 enq 方法将新结点追加到尾结点后
private transient volatile Node tail;

// 自旋和 park 判定的阈值
// The number of times a thread will spin before it gives up and waits for a timeout
// This is useful for avoiding deadlocks, as it ensures that threads will eventually give up and allow other threads to progress.
// This means that a thread will spin 1000 times before it gives up and waits for a timeout. If the thread is still unable to acquire the lock after 1000 spins, it will throw a TimeoutException.
static final long spinForTimeoutThreshold = 1000L;

/**
 * 作为 CAS 操作的支持，虽然使用 AtomicInteger 原子类能获得同样的效果并且可能会达到更好的效果，但是
 * 为了未来的延展性，使用 Unsafe 即可。
 */
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long stateOffset;
private static final long headOffset;
private static final long tailOffset;
private static final long waitStatusOffset;
private static final long nextOffset;

/**
 * 与多线程相关属性的获取通常会使用 unsafe.objectFieldOffset 方法，因为多线程环境下，想要访问或者修
 * 改对象的属性值通常需要加锁来保证线程安全，而 unsafe.objectFieldOffset 方法可以直接获取某个属性值
 * 在内存上的偏移量，避免了对对象的加锁操作，可以提高程序的性能。
 */
static {
    try {
        stateOffset = unsafe.objectFieldOffset
            (AbstractQueuedSynchronizer.class.getDeclaredField("state"));
        headOffset = unsafe.objectFieldOffset
            (AbstractQueuedSynchronizer.class.getDeclaredField("head"));
        tailOffset = unsafe.objectFieldOffset
            (AbstractQueuedSynchronizer.class.getDeclaredField("tail"));
        waitStatusOffset = unsafe.objectFieldOffset
            (Node.class.getDeclaredField("waitStatus"));
        nextOffset = unsafe.objectFieldOffset
            (Node.class.getDeclaredField("next"));

    } catch (Exception ex) { throw new Error(ex); }
}
```

#### 内部方法

```java
// 如果当前 state 的值和期望值相等，使用原子操作更新 state
protected final boolean compareAndSetState(int expect, int update) {
  return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}

// Inserts node into queue, initializing if necessary
private Node enq(final Node node) {}

// Creates and enqueues node for current thread and given mode.
private Node addWaiter(Node mode) {}

// 获取独占锁
// 实现中至少调用一次 tryRelease 方法
// This method can be used to implement method Lock.lock.
public final void acquire(int arg) {
    if (!tryAcquire(arg) && // 尝试获取
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) // 获取失败则入队自旋执行 tryAcquire
        selfInterrupt(); // 在队列中自旋获取失败则将线程中断
}

// 释放独占锁，会忽视线程是否已中断
// This method can be used to implement method Lock.unlock.
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

// Acquires in shared mode, ignoring interrupts.
// 逻辑和 acquire 相似
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

#### 模版方法

```java
/**
 * 尝试获取独占锁，会查询当前 state 是否支持独占模式，如果支持则尝试获取
 * 始终会被当前执行的线程调用，如果获取失败，当前线程可能会进入等待队列。如果未入队，会一直阻塞知道有其
 * 他线程唤醒
 * 可以用来实现 Lock.tryLock() 方法，默认实现抛出 UnsupportedOperationException
 */
protected boolean tryAcquire(int arg) {}

// 尝试释放独占锁，始终会被当前执行的线程调用，默认实现抛出 UnsupportedOperationException
protected boolean tryRelease(int arg) {}

/**
 * 尝试获取共享锁，会查询当前 state 是否支持共享模式，如果支持则尝试获取
 * 始终会被当前执行的线程调用，如果获取失败，当前线程可能会进入等待队列。如果未入队，会一直阻塞知道有其
 * 他线程唤醒
 * 默认实现抛出 UnsupportedOperationException
 */
protected int tryAcquireShared(int arg) {}

// 尝试释放共享锁，始终会被当前执行的线程调用，默认实现抛出 UnsupportedOperationException
protected boolean tryReleaseShared(int arg) {}

// 检查当前是独占还是共享模式
// 仅被 AbstractQueuedSynchronizer.ConditionObject 中的方法调用，如果不用 Condition 则无需实现
// 默认实现抛出 UnsupportedOperationException
protected boolean isHeldExclusively() {}
```



> 这篇[文章](https://mp.weixin.qq.com/s/hvku5GPxkfQ5GffLjiUAYw)中有几个 AQS 节点切换图可以看看。

<br>

### Condition

> 一个 Condition 实例与一个锁绑定，要为一个特定的锁实例获得 Condition 实例，使用对应的 newCondition方法

```java
public interface Condition
```

**内置方法**

```java
/**
 * Causes the current thread to wait until it is signalled or interrupted.
 * The lock associated with this Condition is atomically released.
 * 
 * Some other thread invokes the signal method for this Condition and the current thread happens to be chosen as the thread to be awakened
 * Some other thread invokes the signalAll method for this Condition
 * Some other thread interrupts the current thread, and interruption of thread suspension is supported
 * A "spurious wakeup" occurs
 */
void await() throws InterruptedException;

// Wakes up one waiting thread.
void signal();

// Wakes up all waiting threads.
void signalAll();
```



<br>

### LockSupport

> 用于创建锁和其他同步类的基本线程阻塞原语，底层基于 sun.misc.Unsafe 类的 park 和 unpark 方法。
> 该类与使用它的每个线程相关联，调用 park 方法会阻塞线程；调用 unpark 方法会让线程恢复。

```java
public class LockSupport
```

<br>

**内部属性**

```java
// Unsafe 类辅助实现 park 和 unpark
private static final sun.misc.Unsafe UNSAFE;
```

<br>

**内部方法**

```java
// 阻塞当前线程
// 调用当前当前线程的 unpark 恢复
// 或线程被中断
// 或线程被虚假唤醒
public static void park() {
    UNSAFE.park(false, 0L);
}

// 恢复指定线程
public static void unpark(Thread thread) {
    if (thread != null)
        UNSAFE.unpark(thread);
}
```



<br>

## 锁

### Lock

> Lock 的实现类提供了比 synchronized 方法和 synchronized 块更具扩展性的锁操作。允许更灵活的结构，可能有不同的属性，可能支持多个相关的 Condition 对象。
> 
> 锁是一种用于控制多个线程对共享资源的访问的工具。通常，锁提供对共享资源的排他性访问：一次只有一个线程可以获得锁，对共享资源的访问需要先获得锁。但也有些锁允许对共享资源的并发访问，例如 ReadWriteLock 的读锁。



> 使用 synchronized 方法或 synchronized 代码块可以获取与每个对象相关的隐式监控锁，但锁的获取和释放操作都必须在 synchronized 块内进行。当获取多个锁时，它们必须以相反的顺序释放，并且所有锁必须在获取它们的同一词法范围内释放。
> 
> 虽然 synchronized 方法和语句的范围机制使得监控锁编程的使用变得更加容易，并有助于避免许多涉及锁的常见编程错误，但在某些场合，需要以更灵活的方式使用锁。Lock 接口的实现允许在不同的范围内获取和释放一个锁，并允许以任何顺序获取和释放多个锁。
> 
> 灵活性的增加，也带来了额外的负担。没有 synchronized 块状结构的约束，就没有 synchronized 方法和 synchronized 块自动释放锁的情况。
> 
> ```java
>  Lock l = ...;  
>  l.lock();  
>  try {    
>    // access the resource protected by this lock  
>  } finally {    
>  	l.unlock();  // 因此需要注意手动释放锁资源
>  }
> ```
> 
> 
> 
> Lock 的实现提供了比使用同步方法和语句更多的功能，它提供了获取锁的非阻塞尝试 tryLock，获取可以中断的锁的尝试 lockInterruptibly，以及获取可以超时的锁的尝试 tryLock(long, TimeUnit)

```java
public interface Lock
```

**内置方法**

```java
// Acquires the lock. 如果未获取成功则阻塞直到获取到锁
void lock();

// Acquires the lock unless the current thread is interrupted.
void lockInterruptibly() throws InterruptedException;

// Acquires the lock only if it is free at the time of invocation.
boolean tryLock();

// Acquires the lock if it is free within the given waiting time and the current thread 
// has not been interrupted.
boolean tryLock(long time, TimeUnit unit) throws InterruptedException;

// Releases the lock.
void unlock();

// Returns a new Condition instance that is bound to this Lock instance.
Condition newCondition();
```



<br>

### ReentrantLock

> 一个可重入的互斥锁，其基本行为和语义与使用同步方法和语句获取锁相同，但扩展性更好。一个 ReentrantLock 被最后成功加锁且目前还没有解锁的线程所拥有。
> 
> 如果当前线程已拥有锁，该方法将立即返回。可以通过 isHeldByCurrentThread 和 getHoldCount 方法来检查当前线程是否持有锁资源。
> 
> 对于同一个线程，最多支持 2147483647 次锁的重用，超过此次数将会抛出 Error

> **构造函数**
> 
> 构造函数中接受一个公平性参数，可以为 true 或 false。true 表示获取的是公平锁，反之获取非公平锁。
> 
> 锁的公平性并不能保证线程调度的公平性，许多使用公平锁的线程可能会连续多次获得锁。tryLock 方法并不遵循公平性的规格。如果锁资源空闲，就尝试获取

> **实现**
> 
> 应该总是在 lock 方法之后立即跟着 try-finally 块以进行 unlock 操作。ReentrantLock 除了实现 Lock 接口的方法之外还额外定义了检查和监控 state 状态的 public 和 protected 方法

> **序列化与反序列化**
> 
> 序列化后进行反序列化时，state 永远处于解锁状态

<br>

```java
public class ReentrantLock implements Lock, java.io.Serializable
```

#### 内部类

##### Sync

> ReentrantLock 同步机制控制的基础，使用 AQS 的 state 属性来表示锁被持有的数量。
> 
> 子类分别是 FairSync 和 NonfairSync 用来实现公平和非公平锁。

```java
abstract static class Sync extends AbstractQueuedSynchronizer
```

**内部属性/方法**

```java
// 锁的形式，由子类来实现
abstract void lock();

// 检查锁资源是否由当前线程占有
protected final boolean isHeldExclusively() {}

// 非公平的方式获取锁
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread(); // 获取当前线程
    int c = getState(); // 获取当前 state
    if (c == 0) { // state == 0 表示当前无任何线程占有锁
        if (compareAndSetState(0, acquires)) { // CAS 占有锁
            setExclusiveOwnerThread(current); // 将当前线程设置为占有锁的线程
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) { // 如果锁已经被当前线程占有
        int nextc = c + acquires; // 重入，每次重入 state + 1
        if (nextc < 0) // overflow 达到最大重入次数，抛出 Error
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false; // 当前锁被其他线程占有
}

// 尝试释放锁资源
protected final boolean tryRelease(int releases) {
    int c = getState() - releases; // state 减一
    if (Thread.currentThread() != getExclusiveOwnerThread()) // 检查锁是否被当前线程占有
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) { // state 减到 0 表示锁释放完毕
        free = true;
        setExclusiveOwnerThread(null); // 将占有锁的线程设置为 null
    }
    setState(c);
    return free;
}

// 定义和当前锁相关的 Condition
final ConditionObject newCondition() {
    return new ConditionObject();
}
```

##### NonfairSync

> 非公平的 Sync 子类，不保证锁的获取顺序，并发情况下吞吐量更大
> 
> 
> 
> 非公平锁调用逻辑
> NonfairSync#lock -\> AQS#acquire -\> NonfairSync#tryAcquire -\> Sync#nonfairTryAcquire

```java
static final class NonfairSync extends Sync
```

**内部方法**

```java
// 实现非公平锁
// acquire 方法由 AQS 实现，是一个模版方法，实际上内部调用的是 tryAcquire
final void lock() {
    if (compareAndSetState(0, 1)) // 锁获取成功
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1); //锁获取失败
}
```

<br>

##### FairSync

> 公平的 Sync 子类，保证锁的获取顺序
> 
> 
> 
> 公平锁调用逻辑：
> FairSync#lock -\> AQS#acquire -\> FairSync#tryAcquire

```java
static final class FairSync extends Sync
```

**内部方法**

```java
final void lock() {
    acquire(1);
}

// Fair version of tryAcquire. 
// Don't grant access unless recursive call or no waiters or is first.
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread(); // 获取当前线程
    int c = getState(); // 获取当前 state
    if (c == 0) { // 当前无任何线程占有锁
        if (!hasQueuedPredecessors() && // 队列中没有前驱
            compareAndSetState(0, acquires)) { // 获取锁
            setExclusiveOwnerThread(current); // 将当前线程设置为锁的占有线程
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) { // 重入锁
        int nextc = c + acquires; // state + 1
        if (nextc < 0) // state overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```



<br>

#### 内部方法

```java
// 默认为非公平锁
public ReentrantLock() {
    sync = new NonfairSync();
}

// 获取公平 or 非公平锁
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}

// Acquires the lock 如果已经占有锁，则 state + 1
public void lock() { sync.lock(); }

// Acquires the lock only if it is not held by another thread at the time of 
// invocation.

// 会忽视公平/非公平模式，只要锁空闲就会尝试获取，如果已经占有锁，则 state + 1
// 如果想要遵循公平模式，使用 tryLock(0, TimeUnit.SECONDS)
public boolean tryLock() {
    return sync.nonfairTryAcquire(1);
}

// 如果当前线程占有锁，则 state - 1，当 state 减到 0 时释放锁
// 如果当前线程未占有锁抛出 IllegalMonitorStateException
public void unlock() {
    sync.release(1);
}

// 返回和当前 Lock 实例相关联的 Condition 实例
public Condition newCondition() {
    return sync.newCondition();
}

// 检查是否有在排队获取锁的线程，使用公平锁时会调用到这个方法
// ReentrantLock.FairSync#tryAcquire -> hasQueuedThreads
public final boolean hasQueuedThreads() {
    return sync.hasQueuedThreads();
}

// 检查是否有其他线程在等待同一个 Condition
public boolean hasWaiters(Condition condition) {}
```



<br>

### ReadWriteLock

> ReadWriteLock 维护一对相关的锁，一个读锁，一个写锁。读锁支持多个线程并发访问共享变量；写锁是独占锁。
> 
> 与互斥锁相比，读写锁允许对共享数据的并发读。虽然每次只有一个线程（写线程）可以修改共享数据，但任何数量的线程（读线程）都可以并发地读取数据。
> 
> 使用读写锁是否能比使用互斥锁获得更高的性能，取决于数据的读取和修改的频率、读写操作的时间以及在同一时间试图读取或写入数据的线程数量。如果一个共享变量读取的频率高于写入的频率，则符合读写锁的使用场景。

> **读写偏向策略**
> 
> * 当读写线程都在等待释放锁的时候，授予读锁还是写锁？
> 
>   写优先是很常见的，因为预计写的时间很短，而且不经常发生。读偏好不太常见，如果读线程较频繁且持续时间长，可能会导致长时间的写入延迟。
> 
>   当然也可以选择公平/非公平模式。
> 
> * 确定锁是否是可重入的
> 
>   拥有写锁的线程能否重新获得它？它能否在持有写锁的同时获得一个读锁？读锁本身是可重入的吗？
> 
> * 写锁能否降级为读锁且不允许有写线程介入？一个读锁能否升级为写锁，且优先于其他等待的读或写线程？

> **内存效应**
> 
> 成功获得读锁的线程能看到在释放写锁之前做出的所有更新。

```java
public interface ReadWriteLock
```

```java
// Returns the lock used for reading.
Lock readLock();

// Returns the lock used for writing.
Lock writeLock();
```



<br>

### ReentrantReadWriteLock

> ReadWriteLock 的实现类，具有和 ReentrantLock 相似的语义

> **锁的获取策略**
> 
> ReentrantReadWriteLock 支持公平和非公平模式
> 
> * 非公平模式（默认）
> 
>   忽略获取顺序，锁只要空闲就抢占，比公平模式具有更高的吞吐量
> 
> * 公平模式

> **可重入性**
> 
> ReentrantReadWriteLock 的读写锁都是可重入锁。写线程可以获得读锁，但反之则不能。如果占有了读锁，在获取写锁时需要先释放读锁。
> 
> 当写锁中调用或回调那些在读锁下进行读取的方法时，重入是有用的。换言之，占有写锁的同时可以重入读锁。
> 
> 读锁和写锁最高支持 65535 次重入，超过抛出 Error

> **锁降级**
> 
> 写锁可以通过获取读锁，降级为读锁。先获取写锁，再获取读锁，最后释放写锁，就完成降级。但是从读锁升级为写锁是不被允许的

> **锁获取过程中断**
> 
> 读锁写锁都允许在获取的时候中断

> **Condition 支持**
> 
> 写锁提供了一个 Condition 的实现，其行为方式与 ReentrantLock.newCondition 为 ReentrantLock 提供的 Condition 的行为方式相同。当然，这个 Condition 只能与写锁一起使用。
> 
> 读锁不支持 Condition，默认抛出 UnsupportedOperationException。

> **序列化与反序列化**
> 
> 反序列化得到的锁始终处于解锁状态

```java
public class ReentrantReadWriteLock
        implements ReadWriteLock, java.io.Serializable
```

**Usage**

```java
class CachedData {    
  Object data;    
  volatile boolean cacheValid;    
  final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();      
  void processCachedData() {      
    rwl.readLock().lock();      
    if (!cacheValid) {        
      // Must release read lock before acquiring write lock        			
      rwl.readLock().unlock();        
      rwl.writeLock().lock();        
      try {          
        // Recheck state because another thread might have          
        // acquired write lock and changed state before we did.          
        if (!cacheValid) {            
          	data = ...            
            cacheValid = true;          
        }          
        // Downgrade by acquiring read lock before releasing write lock          
        rwl.readLock().lock();        
      } finally {          
        rwl.writeLock().unlock(); // Unlock write, still hold read        
      }      
    }        
    try {        
      use(data);      
    } finally {        
      rwl.readLock().unlock();      
    }    
  }  
}

class RWDictionary {    
  private final Map<String, Data> m = new TreeMap<String, Data>();    
  private final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();    
  private final Lock r = rwl.readLock();    
  private final Lock w = rwl.writeLock();      
  public Data get(String key) {      
    r.lock();      
    try { 
      return m.get(key); 
    } finally { 
      r.unlock(); 
    }    
  }    
  public String[] allKeys() {      
    r.lock();      
    try { 
      return m.keySet().toArray(); 
    } finally { 
      r.unlock(); 
    }    
  }    
  public Data put(String key, Data value) {      
    w.lock();      
    try { 
      return m.put(key, value); 
    } finally { 
      w.unlock(); 
    }    
  }    
  public void clear() {      
    w.lock();      
    try { 
      m.clear(); 
    } finally { 
      w.unlock(); 
    }    
  }  
}
```

<br>

#### 内部类

##### Sync

> ReentrantReadWriteLock 同步机制的内部实现

```java
abstract static class Sync extends AbstractQueuedSynchronizer
```

**内部类/属性/方法**

```java
// 当前线程持有的可重入读锁的计数，用于实现读锁的重入
// 当线程持有的读锁数为 0 时移除
private transient ThreadLocalHoldCounter readHolds;

/**
 * 缓存最后一个成功获得读锁的线程信息，比如该线程的 id 以及该线程持有的读锁个数
 *
 * 在当前线程是最后一个获取读锁线程的情况下：可以优化读锁的获取和释放过程，避免重复计算读锁的持
 * 有次数，从而提高读锁的性能
 *
 * 存活时间可以超过被缓存的线程，不保存对线程的引用，避免错过垃圾回收，仅保存线程 id 和读锁持
 * 有计数
 */
private transient HoldCounter cachedHoldCounter;

/**
 * 一个读锁计数器，用于记录每个线程持有的读锁数
 * 维护一个 count 计数和一个线程 id，不直接引用线程避免存在垃圾引用
 * 缓存在 cachedHoldCounter 中
 */
static final class HoldCounter {
  int count = 0;
  // Use id, not reference, to avoid garbage retention
  final long tid = getThreadId(Thread.currentThread());
}

static final int SHARED_SHIFT   = 16;

// 读锁计数器和写锁状态存储在同一个 int 类型的变量中
// 高 16 位表示写锁持有计数，低 16 位表示读锁持有计数
// 高 16 位表示数据的前 16 位，低 16 位存储的是数据的后 16 位
static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
// 读锁和写锁的最大数量 65535
static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
// 写锁的状态位,被设置为 (1 << 16)，一个整数的第 17 位表示写锁状态。
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

// Returns the number of shared holds
static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
// Returns the number of exclusive holds
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }

/**
 * 尝试获取锁
 * 1、如果读锁或写锁被占有，获取失败
 * 2、如果重入到最大次数，获取失败
 * 3、否则重入或获取成功，更新 state 以及设置锁的 owner 线程为当前线程
 */
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread(); // 获取当前线程
    int c = getState(); // 获取当前 state
    int w = exclusiveCount(c); // 获取独占计数
    if (c != 0) { // state 非 0，锁已经被占有
        // (Note: if c != 0 and w == 0 then shared count != 0)
        if (w == 0 || current != getExclusiveOwnerThread()) // 锁被占有且是独占锁
            return false; // 获取失败
        if (w + exclusiveCount(acquires) > MAX_COUNT) // 已到达最大获取数量
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire // 重入获取锁
        setState(c + acquires); 
        return true;
    }
  	// state 为 0 则尝试获取锁
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}

// 尝试释放锁
protected final boolean tryRelease(int releases) {
    if (!isHeldExclusively()) // 非当前线程占有，释放失败
        throw new IllegalMonitorStateException();
    int nextc = getState() - releases; // 锁释放后 state 的值
    boolean free = exclusiveCount(nextc) == 0; // 释放后独占计数是否为 0
    if (free) // 释放
        setExclusiveOwnerThread(null);
    setState(nextc);
    return free;
}

/**
 * 轻量级 tryAcquireShared
 * 1、如果写锁被其他线程持有，读锁获取失败
 * 2、否则当前线程就有机会获取读锁，然后检查是否需要排队获取读锁。如果不需要排队，则尝试使用 
 * CAS 获取锁；如果需要排队，则入队阻塞。注意：在这这一步不需要检查可重次数， 
 * fullTryAcquireShared 才需要进行重入次数检查。
 * 3、如果第 2 步因为线程 CAS 获取锁失败或锁被获取的次数达到最大而获取失败，
 * 尝试 fullTryAcquireShared
 */
protected final int tryAcquireShared(int unused) {
    Thread current = Thread.currentThread(); // 获取当前线程
    int c = getState(); // 获取当前 state
    if (exclusiveCount(c) != 0 && // 如果存在独占锁
        getExclusiveOwnerThread() != current) // 且拥有独占锁的线程不是当前线程
        return -1; // 获取共享锁失败
    int r = sharedCount(c); // 持有共享锁的线程数
    if (!readerShouldBlock() && // 检查是否需要阻塞当前线程
        r < MAX_COUNT && // 检查读锁是否达到最大持有数
        compareAndSetState(c, c + SHARED_UNIT)) { // 尝试 CAS 获取读锁
        if (r == 0) { // 当前是第一个持有读锁的线程
            firstReader = current; 
            firstReaderHoldCount = 1;
        } else if (firstReader == current) { // 当前是第一个持有读锁的线程，重入读锁
            firstReaderHoldCount++;
        } else { // 当前线程不是第一个持有读锁的线程
            HoldCounter rh = cachedHoldCounter; // 最后一个成功获取读锁的线程
          	// rh == null 没有线程占有读锁
          	// rh.tid != getThreadId(current) 读锁已被占有，但是不是当前线程
          	// 以下情况可能会走到这一步：当前线程已经获取了写锁，但是需要锁降级成读锁
            if (rh == null || rh.tid != getThreadId(current)) 
                // 获取当前线程的读锁计数，并缓存到 cachedHoldCounter
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0) // rh.count == 0 说明该线程不持有读锁
                readHolds.set(rh); // 将持有读锁的线程信息保存到当前线程的 ThreadLocal
            rh.count++; // 当前线程持有的读锁计数 + 1
        }
        return 1;
    }
  	// 如果上面获取读锁的流程都失败，尝试 fullTryAcquireShared
    return fullTryAcquireShared(current);
}

// 完整版的读锁获取
// 可以处理 tryAcquireShared 中没有处理的 CAS 缺失和读锁重入计数
// 与 tryAcquireShared 中的部分内容重复，但总体上更简单，不需要在重试和惰性读取读锁持有计数
// 之间进行交互
final int fullTryAcquireShared(Thread current) {
    HoldCounter rh = null;
    for (;;) { // 自旋获取读锁
        int c = getState(); // 获取当前 state
        if (exclusiveCount(c) != 0) {
            if (getExclusiveOwnerThread() != current)
                return -1; // 存在独占锁且不是当前线程持有，锁获取失败
            // else we hold the exclusive lock; blocking here
            // would cause deadlock.
        } else if (readerShouldBlock()) { // 如果当前线程持有独占锁，阻塞所有读线程
            // Make sure we're not acquiring read lock reentrantly
            if (firstReader == current) { // 检查是否是第一个读线程
                // assert firstReaderHoldCount > 0;
            } else { // 不是第一个读线程
                if (rh == null) {
                  	// rh 最后一个成功获取读锁的线程持有的读锁计数
                    rh = cachedHoldCounter;
                  	// rh == null 读锁没有被占有
                  	// rh.tid != getThreadId(current) 读锁被占有，但不是当前线程
                    if (rh == null || rh.tid != getThreadId(current)) {
                      	// 获取当前线程的读锁计数
                        rh = readHolds.get();
                        if (rh.count == 0) // rh.count == 0 说明该线程不再持有读锁
                            readHolds.remove(); // 将其移除
                    }
                }
                if (rh.count == 0)
                    return -1;
            }
        }
        if (sharedCount(c) == MAX_COUNT) // 达到最大持有数
            throw new Error("Maximum lock count exceeded");
        if (compareAndSetState(c, c + SHARED_UNIT)) { // CAS 获取读锁
            if (sharedCount(c) == 0) { // 如果当前是第一个占有读锁的线程
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) { // 已经是第一个占有读锁的线程
                firstReaderHoldCount++;
            } else { // 读锁已被其他线程获取
                if (rh == null)
                  	// 获取最后一个成功获取读锁的线程
                    rh = cachedHoldCounter;
                // rh == null 读锁未被占有
              	// rh.tid != getThreadId(current) 读锁被占有但不是当前线程
                if (rh == null || rh.tid != getThreadId(current))
                  	// 获取当前线程的读锁计数
                    rh = readHolds.get();
                else if (rh.count == 0) // 获取读锁
                    readHolds.set(rh);
                rh.count++;
                cachedHoldCounter = rh; // cache for release
            }
            return 1;
        }
    }
}

// 尝试释放读锁
protected final boolean tryReleaseShared(int unused) {
    Thread current = Thread.currentThread(); // 获取当前线程
    if (firstReader == current) { // 如果当前线程是第一个持有读锁的线程
        // assert firstReaderHoldCount > 0;
        if (firstReaderHoldCount == 1) // 如果读锁只被当前线程持有 1 次
            firstReader = null;
        else
            firstReaderHoldCount--; // 读锁被当前线程重入多次
    } else { // 存在其他线程先占有了读锁
      	// 最后一个成功获取到读锁的线程 HoldCounter 缓存
        HoldCounter rh = cachedHoldCounter; 
      	// rh == null 读锁未被占有
        // rh.tid != getThreadId(current) 读锁被占有但不是当前线程
        if (rh == null || rh.tid != getThreadId(current)) // 未持有读锁
          	// 获取当前线程读锁计数
            rh = readHolds.get(); 
        int count = rh.count;
        if (count <= 1) {
            readHolds.remove(); // 释放读锁重入计数
            if (count <= 0)
                throw unmatchedUnlockException();
        }
        --rh.count;
    }
    for (;;) {
        int c = getState();
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc)) // CAS 设置 state
          	// 释放读锁对读线程无任何影响，
          	// 如果读锁和写锁都是空闲的，读线程可能会等待写线程先执行
            return nextc == 0;
    }
}

/**
 * 尝试获取写锁，不遵循公平模式规则，如果锁资源空闲，允许抢占
 * 除了没有调用 writerShouldBlock，其他逻辑和 tryAcquire 一致
 */
final boolean tryWriteLock() {
    Thread current = Thread.currentThread(); // 获取当前线程
    int c = getState(); // 获取当前 state
    if (c != 0) { // state 非 0，表示锁已被占有
        int w = exclusiveCount(c);
        if (w == 0 || current != getExclusiveOwnerThread()) // 是否是当前线程持有
            return false;
        if (w == MAX_COUNT) // 达到最大持有数
            throw new Error("Maximum lock count exceeded");
    }
    if (!compareAndSetState(c, c + 1)) // CAS 获取
        return false;
    setExclusiveOwnerThread(current);
    return true;
}

// 尝试获取读锁
final boolean tryReadLock() {
    Thread current = Thread.currentThread(); // 获取当前线程
    for (;;) {
        int c = getState(); // 获取当前 state
        if (exclusiveCount(c) != 0 && // 是否是独占锁
            getExclusiveOwnerThread() != current) // 独占锁是否由当前线程持有
            return false;
        int r = sharedCount(c); // 共享锁持有计数
        if (r == MAX_COUNT) // 达到最大持有数
            throw new Error("Maximum lock count exceeded");
        if (compareAndSetState(c, c + SHARED_UNIT)) { // CAS 获取读锁
            if (r == 0) { // 如果是第一个占有读锁
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) { // 已经是第一个占有读锁的线程
                firstReaderHoldCount++;
            } else { // 读锁已被其他线程占有
              	// 最后一个成功获得读锁的线程计数
                HoldCounter rh = cachedHoldCounter; 
              	// rh == null 读锁未被占有
              	// rh.tid != getThreadId(current) 读锁被占有，但不是当前线程
                if (rh == null || rh.tid != getThreadId(current))
                  	// 获取当前线程的读锁计数
                    cachedHoldCounter = rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
              	// 以上判断是为了避免同一个线程在未释放之前重复获取读锁
              	// 以及确保当前线程拥有的读锁与试图获取的读锁不冲突
                rh.count++;
            }
            return true;
        }
    }
}

// 供外部类使用，创建和当前锁相关量的 Condition 对象
final ConditionObject newCondition() {
    return new ConditionObject();
}
```

<br>

##### NonfairSync

```java
static final class NonfairSync extends Sync
```

**内部类/属性/方法**

```java
final boolean writerShouldBlock() { // 阻塞写线程
    return false; // writers can always barge
}

// 为了写线程处于无限的等待，如果等待队列头是写线程，则阻塞所有读线程
final boolean readerShouldBlock() { // 阻塞读线程
    return apparentlyFirstQueuedIsExclusive();
}
```

<br>

##### FairSync

```java
static final class FairSync extends Sync
```

```java
final boolean writerShouldBlock() {
    return hasQueuedPredecessors(); // 如果存在等待队列，则阻塞 or 入队写线程
}
final boolean readerShouldBlock() {
    return hasQueuedPredecessors(); // 如果存在等待队列，则阻塞 or 入队读线程
}
```

<br>

##### ReadLock

> **公平读锁调用流程**
> 
> ReentrantReadWriteLock.ReadLock#lock 
> 
> -\> AQS#acquireShared 
> 
> -\> ReentrantReadWriteLock.Sync#tryAcquireShared 
> 
> -\> ReentrantReadWriteLock.Sync#fullTryAcquireShared
> 
> <br>
> 
> **非公平读锁调用流程**
> 
> ReadLock#tryLock
> 
>  -\> ReentrantReadWriteLock.Sync#tryReadLock 

```java
public static class ReadLock implements Lock, java.io.Serializable
```

**内部方法**

```java
public void lock() { sync.acquireShared(1); } // 获取锁

public boolean tryLock() { return sync.tryReadLock(); } // 尝试获取读锁，忽略公平规则

public void unlock() { sync.releaseShared(1); }

// 读锁不支持 Condition
public Condition newCondition() { throw new UnsupportedOperationException(); }
```



<br>

##### WriteLock

> **公平写锁调用流程**
> 
> ReentrantReadWriteLock.WriteLock#lock
> 
> -\> AQS#acquire
> 
> -\> ReentrantReadWriteLock.Sync#tryAcquire
> 
> <br>
> 
> **非公平写锁调用流程**
> 
> ReentrantReadWriteLock.WriteLock#tryLock
> 
>  -\> ReentrantReadWriteLock.Sync#tryWriteLock 

```java
public static class WriteLock implements Lock, java.io.Serializable
```

**内部方法**

```java
public void lock() { sync.acquire(1); } // 获取锁

public boolean tryLock( ) { return sync.tryWriteLock(); } // 尝试获取写锁，忽略公平规则

public void unlock() { sync.release(1); }

// 获取与当前写锁相关联的 Condition 实例
public Condition newCondition() { return sync.newCondition(); }
```



<br>

#### 内部属性

```java
final Sync sync; // 同步机制
private final ReentrantReadWriteLock.ReadLock readerLock; // 读锁
private final ReentrantReadWriteLock.WriteLock writerLock; // 写锁

// Unsafe mechanics
private static final sun.misc.Unsafe UNSAFE; // Unsafe 类
private static final long TID_OFFSET; // 线程 id 的 offset
static {
    try {
        UNSAFE = sun.misc.Unsafe.getUnsafe();
        Class<?> tk = Thread.class;
        TID_OFFSET = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("tid"));
    } catch (Exception e) {
        throw new Error(e);
    }
}
```

<br>

#### 内部方法

```java
// 获取写锁
public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }
// 获取读锁
public ReentrantReadWriteLock.ReadLock  readLock()  { return readerLock; }
```

<br>

#### 总结

> * 读锁不支持 Condition
> * 在只获取一种锁的情况下，读/写锁都是可重入锁。在获取了读锁的情况下，不允许重入写锁；在获取了写锁的情况下，允许重入读锁



<br>

### StampedLock

> 一个基于能力的锁，有三种模式来控制读/写。StampedLock 的状态由 version 和 mode 两部分组成，锁的获取方法返回一个 stamp，代表并控制与锁状态有关的访问。
> 
> 如果使用 tryXXX 方法获取锁，返回 0 表示获取失败。锁的释放和转换方法需要传入 stamp 作为参数，首先进行 stamp 校验，如果 stamp 不匹配则操作失败。

> **三种锁模式**
> 
> * 读锁
> 
>   readLock，可能会阻塞非独占访问，返回一个 stamp 用于 unlockRead 解锁。
> 
> * 写锁
> 
>   writeLock，可能会阻塞独占访问，返回一个 stamp 用于 unlockWrite 解锁。如果写锁被占有，则所有的读锁获取操作都会失败，并且乐观锁校验都会失败。
> 
> * 乐观锁
> 
>   tryOptimisticRead，如果当前不是写模式，则返回非 0 的 stamp 表示获取到乐观锁。在获取到乐观锁后，可以使用 validate 方法检查写锁是否被占有。
> 
>   乐观锁可以认为是一个很弱的读锁，随时都有可能被写锁打破。它只会返回一个 stamp 用作校验，并不会真正上锁不会阻塞其他线程。
> 
>   使用乐观锁对简短的只读代码段通常会降低线程对锁的争抢，提高吞吐量。

> **锁模式转换**
> 
> StampedLock 提供三种锁模式之间的转换，tryConvertToWriteLock 方法可以尝试进行锁升级。
> 
> 如果当前
> 
> * 已经处于写模式
> * 处于读模式，但是没有其他读线程
> * 处于乐观模式，且读写锁都可用
> 
> 都会返回一个有效的 stamp，代表锁转换成功

> **公平与非公平**
> 
> 同样的，StampedLock 提供了 tryXXX 方法，这些方法不遵循公平规则，只要锁处于空闲状态都会尝试去获取。StampedLock 没有内置的等待队列，而是通过尝试获取读锁或写锁来实现并发控制，不支持 Condition

> **可重入性**
> 
> StampedLock 不可重入，在使用 StampedLock 的时候，如果一个线程已经持有了写锁，则其它线程就不能获取读锁或者写锁，直到持有写锁的线程释放锁。如果一个线程已经持有了写锁，再次尝试获取写锁会导致死锁。如果一个线程已经持有了读锁，再次尝试获取写锁也会导致死锁。
> 
> 尽管可以将 stamp 作为参数传入 tryConvertToWriteLock 尝试进行锁的转换，但是重入是不被允许的。

> **Stamp 取值**
> 
> Stamp 的取值是有限的，在密码学上是不安全的，一个有效的 stamp 是有可能被猜到的。stamp 在运行一年之后可以进行回收，超过这段时间没有使用或者进行校验的 stamp 可能会无法正确校验。

> **序列化与反序列化**
> 
> StampedLock 是可以序列化的，但总是反序列化为无锁状态，所以在远程调用中是不可用的

> **锁的状态表示**
> 
> 通常来说，锁的主要状态主要由二进制串表示，初始值为 256，二进制 100000000，二进制下标位从 0 开始，从右往左第 7 位（左高右低）为 0 表示锁未被占有。
> 
> 当锁第一次被获取 stamp = state + WBIT = 256 + 128 = 384，第一次获取锁时返回的 stamp = 384，从右往左第 7 位为 1，表示锁被占有。
> 
> 此二进制序列号会被读锁的非零计数抵消，如果获取的是乐观锁，读计数会忽略。因为需要用一个足够小的有限数来表示读计数，读锁超过计数最大值时，使用 RBITS 来表示读锁溢出，作为更新溢出的自旋保护机制

```java
public class StampedLock implements java.io.Serializable
```

**Usage**

```java
class Point {    
  private double x, y;    
  private final StampedLock sl = new StampedLock();
  
  void set(double deltaX, double deltaY) { 
    // an exclusively locked method      
    long stamp = sl.writeLock();      
    try {        
      // do something     
    } finally {        
      sl.unlockWrite(stamp);      
    }    
  }
  
  double get() { 
    // A read-only method      
    long stamp = sl.tryOptimisticRead(); // get  OptimisticRead first
    double currentX = x, currentY = y;      
    if (!sl.validate(stamp)) { // check OptimisticRead stamp     
      stamp = sl.readLock();         
      try {           
        // read data     
      } finally {            
        sl.unlockRead(stamp);         
      }      
    }      
    return data;  
  }
  
  // upgrade      
  void upgrade(double newX, double newY) { 
    // Could instead start with optimistic, not read mode      
    long stamp = sl.readLock();      
    try {        
      while (x == 0.0 && y == 0.0) {          
        long ws = sl.tryConvertToWriteLock(stamp); // upgrade read to write        
        if (ws != 0L) {            
          stamp = ws;            
          x = newX;            
          y = newY;            
          break;          
        }          
        else { // upgrade fail, relase read and acquire write
          sl.unlockRead(stamp);            
          stamp = sl.writeLock();          
        }        
      }      
    } finally {        
      sl.unlock(stamp);      
    }    
  }  
}
```

#### 内部类

##### WNode

```java
static final class WNode {
    volatile WNode prev;
    volatile WNode next;
    volatile WNode cowait;    // list of linked readers // 读等待列表
    volatile Thread thread;   // non-null while possibly parked
    volatile int status;      // 0, WAITING, or CANCELLED
    final int mode;           // RMODE or WMODE
    WNode(int m, WNode p) { mode = m; prev = p; }
}
```



<br>

##### ReadLockView

> Returns a ReadWriteLock view of this StampedLock in which the ReadWriteLock.readLock() method is mapped to asReadLock()

```java
final class ReadLockView implements Lock
```

```java
public void lock() { readLock(); } // StampedLock#readLock
public boolean tryLock() { return tryReadLock() != 0L; } // StampedLock#tryReadLock
public void unlock() { unstampedUnlockRead(); } // StampedLock#unstampedUnlockRead

// 不支持 Condition
public Condition newCondition() { throw new UnsupportedOperationException(); }
```

<br>

##### WriteLockView

> Returns a ReadWriteLock view of this StampedLock in which the ReadWriteLock.writeLock() is mapped to asWriteLock().

```java
final class WriteLockView implements Lock
```

```java
public void lock() { writeLock(); } // StampedLock#writeLock
public boolean tryLock() { return tryWriteLock() != 0L; }//StampedLock#tryWriteLock
public void unlock() { unstampedUnlockWrite(); } //StampedLock#unstampedUnlockWrite

// 不支持 Condition
public Condition newCondition() { throw new UnsupportedOperationException(); }
```

<br>

##### ReadWriteLockView

```java
final class ReadWriteLockView implements ReadWriteLock {
    public Lock readLock() { return asReadLock(); }
    public Lock writeLock() { return asWriteLock(); }
}
```

<br>

#### 内部属性

```java
// StampedLock 内部使用了一个 long 类型的变量 state 来记录锁的状态，
// 其中高 16 位表示写锁状态，低 16 位表示读锁状态。
// 其中，LG_READERS 定义了低 16 位中用于记录读锁个数的位数，
// 实际上这个位数就是 16 减 LG_READERS，可以用来控制读锁的数量

// 记录读锁个数的位数
// LG_READERS 的默认值是 7，意味着可以支持最多同时持有 2^(16-7) = 256 个读锁
private static final int LG_READERS = 7;

// 表示锁的状态，和计算 stamp 的值

private static final long RUNIT = 1L; // 读锁计数器的单位值，用于读锁计数器的自增

// 写锁的掩码值，用于标识写锁的状态，通过左移操作得到
// 用于检测写锁是否空闲
private static final long WBIT  = 1L << LG_READERS; // 2^7 = 128

// 读锁的掩码值
// 当 state 中低 LG_READERS 位的 RBITS 全部为 1 时，表示当前所有 reader 位都被占用
// 用于检测读锁是否空闲
private static final long RBITS = WBIT - 1L; // 128 - 1 = 127

// 当读锁计数器达到该值时，代表读锁已满，不能再获取新的读锁，用于限制读锁计数器的最大值
private static final long RFULL = RBITS - 1L; // 127 - 1 = 126

// ABITS 和 SBITS 是用来计算锁的状态的，锁的状态是一个 long 类型的值，
// 高 16 位是读锁的计数器，低 16 位是写锁的计数器，中间的1位是一个标志位，用来表示锁的状态。

// Acquiring Bits 获取状态位，当一个线程试图获取一个锁时，被设置为 1
// 用于检测锁是否空闲
private static final long ABITS = RBITS | WBIT; // 127 ｜ 126 = 255 // 11111111

// Successful Bits 成功获取位，当线程获得锁时，它被设置为 1
// 还可以获取乐观锁，校验乐观锁等
private static final long SBITS = ~RBITS; // note overlap with ABITS // -128 // 10000000

// 锁状态的初始值
private static final long ORIGIN = WBIT << 1; // 128 * 2 = 256

// 节点状态值
private static final int WAITING   = -1;
private static final int CANCELLED =  1;
// 节点模式
private static final int RMODE = 0;
private static final int WMODE = 1;
```

#### 内部方法

```java
// 提供给 ReadLockView 和 WriteLockView 使用
// 因为 View 类不会保存 state 信息，所以需要在外部定义

// 无需 stamp 参数释放写锁
final void unstampedUnlockWrite() {
    WNode h; // 头节点
  	long s; // 当前 state
  	// WBIT 写锁对应的状态位
  	// ((s = state) & WBIT) == 0L 读取当前状态 state 并检查写锁是否被持有
  	// 如果当前状态的写锁位为 0，则表示写锁未被持有
    if (((s = state) & WBIT) == 0L)
        throw new IllegalMonitorStateException();
  	// state = (s += WBIT) == 0L 如果写锁位加上 state 后结果为 0
  	// 将状态 state 重置为初始状态 ORIGIN
    state = (s += WBIT) == 0L ? ORIGIN : s;
  	// 释放等待队列中的节点
    if ((h = whead) != null && h.status != 0)
        release(h);
}

// 无需 stamp 参数释放读锁
final void unstampedUnlockRead() {
    for (;;) {
      	// m 表示当前状态下读锁的数量
        long s, m; 
      	WNode h;
      	// 如果 m 的值为 0，或者大于等于 WBIT（写锁被持有）
        if ((m = (s = state) & ABITS) == 0L || m >= WBIT)
            throw new IllegalMonitorStateException();
        else if (m < RFULL) { // 如果读锁的数量小于 RFULL，减少读锁的数量
          	// 尝试使用 CAS 操作将 state 的值减去 RUNIT，表示释放一个读锁
            if (U.compareAndSwapLong(this, STATE, s, s - RUNIT)) {

              	// m == RUNI 仅有一个读锁
              	// (h = whead) != null 存在下一个需要获取锁的节点
              	// h.status != 0 则 status 可能为 WAITING = -1 或 CANCELLED = 1
                if (m == RUNIT && (h = whead) != null && h.status != 0)
                    release(h); // 将 WAITING/CANCELLED 状态的头节点移除避免死锁
                break;
            }
        }
      	// // 如果读锁的数量已经达到了 RFULL 尝试减少读溢出数量
        else if (tryDecReaderOverflow(s) != 0L)
            break;
    }
}

/**
 * 唤醒 h 结点的后继结点（h 通常表示等待结点的头结点），此方法等效于 h.next，
 * 但是它可以在 h 指向的下一个结点有延迟的时候从尾结点遍历，唤醒其他结点
 *
 * 如果后继的一个或者多个线程结点被取消，唤醒可能会失败
 */
private void release(WNode h) {
    if (h != null) {
      	// q 表示当前节点的下一个结点
        WNode q; 
      	Thread w;
      	// 如果 h 结点处于 WAITING 状态，将其状态设置为 0，退出 WAITING
        U.compareAndSwapInt(h, WSTATUS, WAITING, 0);
      	// 如果不存在下一个结点或下一个结点被取消
        if ((q = h.next) == null || q.status == CANCELLED) {
          	// 从尾结点遍历，获取其他等待获取锁的线程
            for (WNode t = wtail; t != null && t != h; t = t.prev)
                if (t.status <= 0) // WAITING = -1 / CANCELLED = 1 如果存在等待结点
                    q = t;
        }
      	// 等待获取锁的结点非空，且结点中的线程依然存活
        if (q != null && (w = q.thread) != null)
            U.unpark(w); // 将其唤醒
    }
}

public long writeLock() { // 在完全无锁的情况下才能成功获取到写锁
    long s, next;  // bypass acquireWrite in fully unlocked case only
    // state=256, WBIT=128, ABITS=255, STATE=16 表示内存偏移量
    // state & ABITS == 0L 表示锁空闲
    // 如果锁没有被获取，并且 CAS 获取锁成功 返回 next = s + WBIT
    // 第一次成功获取返回 stamp=256+128=384
    // 锁获取失败，进入 acquireWrite，在循环中自旋
    return ((((s = state) & ABITS) == 0L &&
             U.compareAndSwapLong(this, STATE, s, next = s + WBIT)) ?
            next : acquireWrite(false, 0L));
}

/**
 * 如果当前锁资源没有被别的线程占有，可以直接获取；
 * 如果当前锁资源被别的线程占有，将当前线程加入等待队列。
 * 如果等待队列中只有一个结点，头结点一直自旋，直到获取到锁或者达到头结点自旋上限；
 * 如果等待队列中存在多个结点，唤醒等待队列中的其他结点。
 */
private long acquireWrite(boolean interruptible, long deadline) {
    // 初始化两个结点，node 表示当前线程
    WNode node = null, p;
    for (int spins = -1;;) { // spin while enqueuing 入队自旋
        long m, s, ns;
        // state & ABITS == 0L 表示锁空闲，直接获取
        if ((m = (s = state) & ABITS) == 0L) {
            if (U.compareAndSwapLong(this, STATE, s, ns = s + WBIT))
                return ns; // 获取成功返回 stamp，退出自旋循环
        }
      	// spins < 0 初始化 spins，设置当前线程最大自旋次数 SPINS= 64
        else if (spins < 0) 
            spins = (m == WBIT && wtail == whead) ? SPINS : 0;
        else if (spins > 0) {
            // 获取伪随机数，如果大于 0 则 spins 自减
            if (LockSupport.nextSecondarySeed() >= 0) 
                --spins;
        }
        // spins 自减到 0 也没能获取到锁，进入下一个 else if
        // p = wtail == null 队列尾结点为空，表示此时队列为空队列，开始初始化队列
        else if ((p = wtail) == null) { // initialize queue
            // 初始化写模式头结点
            // 头结点 h 并不是一个真正的线程节点，它只是一个哨兵结点，用于协调多个线程之间的并发竞争。
            // 在头结点自旋等待获取锁时，实际上是在等待其他线程释放锁，当获取到锁资源时，线程会从等待队列中取
            // 出下一个真正的线程结点 np，将其设置为头结点，并尝试获取锁资源。
            WNode hd = new WNode(WMODE, null); // 需要一个 dummy head 来辅助
            // CAS 操作设置头结点
            if (U.compareAndSwapObject(this, WHEAD, null, hd))
                wtail = hd; // 队列此时只有一个结点，因此尾结点=头结点
        }
      	// 到这一步，说明队列非空，头结点 p 已存在
        else if (node == null) // node == null 初始化当前 node
            // 初始化写模式的 wnode 结点，前驱结点为尾结点 p
            node = new WNode(WMODE, p);
        else if (node.prev != p) // 如果 node 不为 null
            node.prev = p; // node 的前驱设置为尾结点 p
        else if (U.compareAndSwapObject(this, WTAIL, p, node)) { //node设置为尾结点
            p.next = node; // 将尾结点的后继结点设置为 node
            break; // node 结点成功进入等待队列，退出循环
        }
    }

    // 若等待队列已存在 
    // node 始终是尾结点

    // 到这一步：
    // 当前线程自旋获取锁失败，创建 node 结点，作为等待链表的尾结点
    
    // 等待链表情况如下：
    // 1、只有一个结点在等待获取锁 whead = p <-> node = wtail
    // 2、链表已存在，有多个结点，尾结点 node = wtail，p 是 node 的前驱 whead <-> ... <-> p <-> node

    // 头结点自旋获取锁
    for (int spins = -1;;) {
        // 初始化一系列结点
      	 // h 表示头结点
      	 // np 表示 node 的前驱
      	 // pp 表示 p 的前驱
        WNode h, np, pp; int ps;
        if ((h = whead) == p) { // 头结点存在，且头节点 == 尾节点 p 对应： h=whead=p <-> node
            if (spins < 0)
                // HEAD_SPINS 表示头结点最大自旋次数，HEAD_SPINS 和 CPU 核心数有关
              	 // 假设是多核 CPU，HEAD_SPINS=2^10=1024，否则 HEAD_SPINS=0
                spins = HEAD_SPINS;
          	// 多核 CPU，MAX_HEAD_SPINS=2^16=65536，否则 MAX_HEAD_SPINS=0
            else if (spins < MAX_HEAD_SPINS) 
                spins <<= 1; // spins = spins << 1 = spins * 2
            for (int k = spins;;) { // spin at head 头结点自旋
                long s, ns;
                // 尝试获取锁
                if (((s = state) & ABITS) == 0L) { // 获取成功
                    if (U.compareAndSwapLong(this, STATE, s, ns = s + WBIT)) {
                        whead = node; // 头结点自旋获取锁成功，头结点 whead = node
                        node.prev = null;
                        return ns; // 返回更新后的 state
                    }
                }
                // 获取失败，继续自旋
                else if (LockSupport.nextSecondarySeed() >= 0 && --k <= 0)
                    break;
            }
        }
        
        // 头结点自旋获取锁失败

        // 到这一步
        // 头结点自旋超过最大次数，锁未获取成功 h = whead = p
        
        // 等待链表情况如下
        // 情况1：只有一个结点在等待获取锁 h = whead = p <-> node = wtail
        // 情况2：链表已存在，有多个结点，h = whead <-> ... <-> p <-> node
        
        // 唤醒头结点的后继结点（处理情况 2）
        else if (h != null) { // 唤醒链表中的其他等待结点
            WNode c; Thread w;
            // c = h.cowait，c 表示等待获取锁的结点链表，如果等待链表非空，循环唤醒链表中的结点
            while ((c = h.cowait) != null) { // 对应情况 2，等待链表已存在，循环唤醒后继结点
                if (U.compareAndSwapObject(h, WCOWAIT, c, c.cowait) && (w = c.thread) != null)
                    U.unpark(w); // 唤醒后继结点
            }
        }

        // 到这里链表的有两种情况： 
        // 1、h = whead = p <-> node 
        // 2、h = whead <-> ... <-> p <-> node 

        if (whead == h) {
            // np 表示 node 结点的前驱结点
            if ((np = node.prev) != p) {
                if (np != null)
                    // 将 node 放到链表末尾
                    (p = np).next = node;   // stale
            }
            // 将 p 结点的状态设置为 WAITING = -1
            else if ((ps = p.status) == 0)
                U.compareAndSwapInt(p, WSTATUS, 0, WAITING);
            else if (ps == CANCELLED) { // 如果 p 结点状态为 CANCELLED
                // 将 node 结点和 p 的前驱结点 pp 相连接
                if ((pp = p.prev) != null) {
                    node.prev = pp;
                    pp.next = node;
                }
            }
            // 走到这里说明头结点 p 状态已经是 WAITING
            else {
                long time; // 0 argument to park means no timeout
                // deadline = 0
                if (deadline == 0L)
                    time = 0L;
                // deadline 已经过期，取消等待
                else if ((time = deadline - System.nanoTime()) <= 0L)
                    return cancelWaiter(node, node, false);
                // deadline !=0 且未过期
                // 获取当前线程
                Thread wt = Thread.currentThread();
                // 保存对 wt 对象的引用
                U.putObject(wt, PARKBLOCKER, this);
                // 将当前线程保存到 node.thread
                node.thread = wt;
                // p != h ==> p 不是头结点
                // node 的前驱结点是 p 结点
                // whead == h ==> h 是头结点
                // p.status < 0 说明 p 结点在 WAITING 状态
                // state & ABITS != 0L 表示目前的锁已经被其他线程占有
                if (p.status < 0 && (p != h || (state & ABITS) != 0L) &&
                        whead == h && node.prev == p)
                    // 进入这一步说明等待队列中存在多个等待结点

                    // UnSafe.park 模拟 LockSupport.park
                    // 实际上 LockSupport.park 底层实现就是依靠 UnSafe.park阻塞当前线程
                    // 当 balancing unpark 发生或 balancing unpark 已经发生恢复线程被中断后恢复
                    // 如果 absolute=false 并且 time 非零，等待 time 时间后恢复；
                    // 如果 absolute=true 并且 time 非零；或参数 time 过期，线程恢复或虚假恢复

                    // park 当前线程，当前线程进入等待状态，并等待被唤醒
                    U.park(false, time);  // emulate LockSupport.park

                // 到这里说明线程被唤醒，将 node.thread 设置为 null
                // 因为 node 结点是 dummy 结点，因此不会对线程有影响
                node.thread = null;
                // 清空对 wt 对象的引用
                U.putObject(wt, PARKBLOCKER, null);
                if (interruptible && Thread.interrupted())
                    // 如果线程可中断或者已经被中断，取消等待
                    return cancelWaiter(node, node, true);
            }
        }
    }
}

public long tryWriteLock() {
    long s, next;
    // ((s = state) & ABITS) == 0L 读写锁均空闲
    return ((((s = state) & ABITS) == 0L &&
             U.compareAndSwapLong(this, STATE, s, next = s + WBIT)) ?
            next : 0L);
}

public void unlockWrite(long stamp) {
    WNode h;
    // state != stamp stamp 不匹配
    // (stamp & WBIT) == 0L 或写锁处于空闲
    if (state != stamp || (stamp & WBIT) == 0L)
        throw new IllegalMonitorStateException();
    state = (stamp += WBIT) == 0L ? ORIGIN : stamp;
    if ((h = whead) != null && h.status != 0)
        release(h);
}

public long readLock() {
    long s = state, next;  // bypass acquireRead on common uncontended case
    // (whead == wtail && (s & ABITS) < RFULL 读锁持有数为达到最大值
    return ((whead == wtail && (s & ABITS) < RFULL &&
             U.compareAndSwapLong(this, STATE, s, next = s + RUNIT)) ?
            next : acquireRead(false, 0L));
}

private long acquireRead(boolean interruptible, long deadline) {
    // node 表示当前线程结点
    // p 表示尾结点
    WNode node = null, p;
    for (int spins = -1;;) {
        WNode h;
        // 如果头尾结点相等 h = whead = p = wtail 表示当前只有 node 等待获取
        if ((h = whead) == (p = wtail)) {
            for (long m, s, ns;;) {
                // (m = (s = state) & ABITS) < RFULL 表示读锁持有数未达到最大值
                // 则尝试 CAS 获取读锁
                if ((m = (s = state) & ABITS) < RFULL ?
                    U.compareAndSwapLong(this, STATE, s, ns = s + RUNIT) :
                    // m < WBIT 表示当前存在读锁
                    // 如果读锁持有超过最大持有数，先尝试获取锁，成功后增大最大持有数
                    // ++readerOverflow
                    (m < WBIT && (ns = tryIncReaderOverflow(s)) != 0L))
                    return ns;
                else if (m >= WBIT) { // 当前存在写锁
                    if (spins > 0) {
                        if (LockSupport.nextSecondarySeed() >= 0) // 自旋获取读锁
                            --spins;
                    }
                    else {
                        if (spins == 0) { // 自旋结束还是没获取到锁
                            WNode nh = whead, np = wtail;
                            // h = whead
                            // p = wtail
                            if ((nh == h && np == p) || (h = nh) != (p = np))
                                break;
                        }
                        spins = SPINS;
                    }
                }
            }
        }
        
        // 到这里，说明自旋获取读锁未成功
        
        if (p == null) { // initialize queue 初始化队列
            WNode hd = new WNode(WMODE, null); // 创建 dummy head
            if (U.compareAndSwapObject(this, WHEAD, null, hd))
                wtail = hd;
        }
        else if (node == null)
            node = new WNode(RMODE, p); // 创建 node，前驱 p = wtail
        else if (h == p || p.mode != RMODE) {
            if (node.prev != p)
                node.prev = p;
            else if (U.compareAndSwapObject(this, WTAIL, p, node)) {
                p.next = node;
                break;
            }
        }
        // CAS 设置 cowait，即将 node 加入等待队列
        else if (!U.compareAndSwapObject(p, WCOWAIT, node.cowait = p.cowait, node))
            node.cowait = null;
        else {
            // 到这里说明 node 已经在队列中
            for (;;) {
                // pp 表示 p 的前驱
                // c = h.cowait
                WNode pp, c; Thread w;
                if ((h = whead) != null && (c = h.cowait) != null &&
                    U.compareAndSwapObject(h, WCOWAIT, c, c.cowait) &&
                    (w = c.thread) != null) // help release
                    U.unpark(w); // 唤醒 h = whead 的后继读结点
                if (h == (pp = p.prev) || h == p || pp == null) {
                    long m, s, ns;
                    do { // 尝试获取读锁
                        if ((m = (s = state) & ABITS) < RFULL ?
                            U.compareAndSwapLong(this, STATE, s,
                                                 ns = s + RUNIT) :
                            (m < WBIT &&
                             (ns = tryIncReaderOverflow(s)) != 0L))
                            return ns;
                    } while (m < WBIT);
                }
                if (whead == h && p.prev == pp) {
                    long time;
                    // 抛弃已取消的结点
                    if (pp == null || h == p || p.status > 0) { // p.status = CANCELLED = 1 > 0
                        node = null; // throw away
                        break;
                    }
                    if (deadline == 0L)
                        time = 0L;
                    else if ((time = deadline - System.nanoTime()) <= 0L)
                        return cancelWaiter(node, p, false);
                    Thread wt = Thread.currentThread();
                    U.putObject(wt, PARKBLOCKER, this);
                    node.thread = wt;
                    if ((h != pp || (state & ABITS) == WBIT) &&
                        whead == h && p.prev == pp)
                        U.park(false, time); // 还是没获取到锁，park 当前线程 time 时间
                    node.thread = null;
                    U.putObject(wt, PARKBLOCKER, null);
                    if (interruptible && Thread.interrupted())
                        return cancelWaiter(node, p, true);
                }
            }
        }
    }

    for (int spins = -1;;) {
        WNode h, np, pp; int ps;
        if ((h = whead) == p) {
            if (spins < 0)
                spins = HEAD_SPINS;
            else if (spins < MAX_HEAD_SPINS)
                spins <<= 1;
            for (int k = spins;;) { // spin at head 头结点自旋获取锁
                long m, s, ns;
                if ((m = (s = state) & ABITS) < RFULL ?
                    U.compareAndSwapLong(this, STATE, s, ns = s + RUNIT) :
                    (m < WBIT && (ns = tryIncReaderOverflow(s)) != 0L)) {
                    WNode c; Thread w;
                    whead = node;
                    node.prev = null;
                    while ((c = node.cowait) != null) {
                        if (U.compareAndSwapObject(node, WCOWAIT,
                                                   c, c.cowait) &&
                            (w = c.thread) != null)
                            U.unpark(w);
                    }
                    return ns;
                }
                else if (m >= WBIT &&
                         LockSupport.nextSecondarySeed() >= 0 && --k <= 0)
                    break;
            }
        }
        else if (h != null) {
            WNode c; Thread w;
            while ((c = h.cowait) != null) {
                if (U.compareAndSwapObject(h, WCOWAIT, c, c.cowait) &&
                    (w = c.thread) != null)
                    U.unpark(w);
            }
        }
        if (whead == h) {
            if ((np = node.prev) != p) {
                if (np != null)
                    (p = np).next = node;   // stale 唤醒队列中后继结点
            }
            else if ((ps = p.status) == 0)
                U.compareAndSwapInt(p, WSTATUS, 0, WAITING);
            else if (ps == CANCELLED) {
                if ((pp = p.prev) != null) {
                    node.prev = pp;
                    pp.next = node;
                }
            }
            else {
                long time;
                if (deadline == 0L)
                    time = 0L;
                else if ((time = deadline - System.nanoTime()) <= 0L)
                    return cancelWaiter(node, node, false);
                Thread wt = Thread.currentThread();
                U.putObject(wt, PARKBLOCKER, this);
                node.thread = wt;
                if (p.status < 0 &&
                    (p != h || (state & ABITS) == WBIT) &&
                    whead == h && node.prev == p)
                    U.park(false, time);
                node.thread = null;
                U.putObject(wt, PARKBLOCKER, null);
                if (interruptible && Thread.interrupted())
                    return cancelWaiter(node, node, true);
            }
        }
    }
}

public long tryReadLock() {
    for (;;) {
        long s, m, next;
        if ((m = (s = state) & ABITS) == WBIT)
            return 0L;
        else if (m < RFULL) { // 读锁持有数未满
            if (U.compareAndSwapLong(this, STATE, s, next = s + RUNIT))
                return next;
        }
        else if ((next = tryIncReaderOverflow(s)) != 0L) // 读锁持有数已满，尝试自增
            return next;
    }
}

public void unlockRead(long stamp) {
    long s, m; WNode h;
    for (;;) {
        // 满足条件释放读锁
        if (((s = state) & SBITS) != (stamp & SBITS) ||
            (stamp & ABITS) == 0L || (m = s & ABITS) == 0L || m == WBIT)
            throw new IllegalMonitorStateException();
        if (m < RFULL) { // 读锁持有数未满，正常释放
            if (U.compareAndSwapLong(this, STATE, s, s - RUNIT)) {
                if (m == RUNIT && (h = whead) != null && h.status != 0)
                    release(h);
                break;
            }
        }
        else if (tryDecReaderOverflow(s) != 0L) // 读锁持有数超过最大值，尝试减少
            break;
    }
}

// 尝试获取乐观锁
public long tryOptimisticRead() {
    long s;
    return (((s = state) & WBIT) == 0L) ? (s & SBITS) : 0L;
}

// 验证乐观锁 stamp
public boolean validate(long stamp) {
    U.loadFence();
    return (stamp & SBITS) == (state & SBITS);
}

// 解锁
public void unlock(long stamp) {
    // a = stamp & ABITS a 表示锁的状态位
    long a = stamp & ABITS, m, s; WNode h;
    while (((s = state) & SBITS) == (stamp & SBITS)) { // 判断当前锁的状态是否等于给定 stamp 的状态
        if ((m = s & ABITS) == 0L) // 当前锁未被持有
            break;
        else if (m == WBIT) { // 当前锁被写线程持有
            if (a != m) // 校验写线程的标志位是否等于给定的标志位
                break;
            state = (s += WBIT) == 0L ? ORIGIN : s; // 更新锁的状态
            if ((h = whead) != null && h.status != 0) // 判断是否存在等待结点，存在则释放等待结点
                release(h);
            return;
        }
        else if (a == 0L || a >= WBIT) // 给定标志位为 0 或大于等于 WBIT，即非读锁标志位
            break;
        else if (m < RFULL) { // 当前锁被读线程持有
            if (U.compareAndSwapLong(this, STATE, s, s - RUNIT)) { // 尝试释放读锁
                if (m == RUNIT && (h = whead) != null && h.status != 0) // 判断是否存在等待结点，存在则释放等待结点
                    release(h);
                return;
            }
        }
        else if (tryDecReaderOverflow(s) != 0L) // 尝试释放读锁并减少溢出
            return;
    }
    throw new IllegalMonitorStateException();
}

// 尝试转换成写锁
public long tryConvertToWriteLock(long stamp) {
    // a = stamp & ABITS a 表示锁的状态位
    long a = stamp & ABITS, m, s, next;
    while (((s = state) & SBITS) == (stamp & SBITS)) {
        if ((m = s & ABITS) == 0L) { // 当前锁空闲
            if (a != 0L) // 锁状态位不为 0，表示锁已经被获取
                break;
            if (U.compareAndSwapLong(this, STATE, s, next = s + WBIT)) // 直接获取写锁
                return next;
        }
        else if (m == WBIT) { // 当前已经持有写锁
            if (a != m)
                break;
            return stamp;
        }
        else if (m == RUNIT && a != 0L) { // 当前持有读锁
            if (U.compareAndSwapLong(this, STATE, s,
                                     next = s - RUNIT + WBIT)) // 释放读锁获取写锁
                return next;
        }
        else
            break;
    }
    return 0L;
}
// 尝试转换成读锁
public long tryConvertToReadLock(long stamp) {
    // a = stamp & ABITS a 表示锁的状态位
    long a = stamp & ABITS, m, s, next; WNode h;
    while (((s = state) & SBITS) == (stamp & SBITS)) { // 判断当前锁的状态是否等于给定 stamp 的状态
        if ((m = s & ABITS) == 0L) { // 锁空闲，直接获取
            if (a != 0L)
                break;
            else if (m < RFULL) { // 读锁持有数未达到最大
                if (U.compareAndSwapLong(this, STATE, s, next = s + RUNIT))
                    return next;
            }
            else if ((next = tryIncReaderOverflow(s)) != 0L) // 读锁持有数溢出，尝试增加并获取读锁
                return next;
        }
        else if (m == WBIT) { // 当前持有写锁
            if (a != m)
                break;
            // s + (WBIT + RUNIT) 可以看成 (s + WBIT) + RUNIT 先释放写锁在获取读锁
            state = next = s + (WBIT + RUNIT);
            if ((h = whead) != null && h.status != 0) // 判断是否存在等待结点，存在则释放等待结点
                release(h);
            return next;
        }
        else if (a != 0L && a < WBIT)
            return stamp;
        else
            break;
    }
    return 0L;
}
// 尝试转换成乐观锁
public long tryConvertToOptimisticRead(long stamp) {
    // a = stamp & ABITS a 表示锁的状态位
    long a = stamp & ABITS, m, s, next; WNode h;
    U.loadFence(); // 加载屏障
    for (;;) {
        if (((s = state) & SBITS) != (stamp & SBITS)) // 判断当前锁的状态是否等于给定 stamp 的状态
            break;
        if ((m = s & ABITS) == 0L) { // 当前锁空闲
            if (a != 0L) // 锁被抢占，无法转换成乐观锁
                break;
            return s; // 否则获取乐观锁成功
        }
        else if (m == WBIT) { // 当前线程持有写锁
            if (a != m) // 无法转换成乐观锁
                break;
            // s = s + WBIT 释放写锁，返回解锁后的 s 作为乐观锁的 stamp
            state = next = (s += WBIT) == 0L ? ORIGIN : s;
            if ((h = whead) != null && h.status != 0) // 判断是否存在等待结点，存在则释放等待结点
                release(h);
            return next;
        }
        // a == 0L 表示当前线程没有获取到锁
        // a >= WBIT 表示当前存在写锁
        // 转换成乐观锁失败
        else if (a == 0L || a >= WBIT)
            break;
        else if (m < RFULL) { // 当前持有读锁，释放读锁，获取乐观锁
            if (U.compareAndSwapLong(this, STATE, s, next = s - RUNIT)) {
                if (m == RUNIT && (h = whead) != null && h.status != 0)
                    release(h);
                return next & SBITS;
            }
        }
        else if ((next = tryDecReaderOverflow(s)) != 0L)
            return next & SBITS;
    }
    return 0L;
}
```

<br>

## 工具类


<br>

### Semaphore
> 计数信号量，一个信号量维护一组 permit。如果 permit 不可用，调用 acquire 会阻塞线程，直到信号量可用；每次调用 release 方法都会增加 permit。需要注意的是，并没有实际存在的 permit 对象，Semaphore 只是维护了一个计数器代表 permit（实际上就是基于 AQS的 state）。
> 
> 信号量通常用于限制可以访问某些资源的线程数。一个初始值为 1 的信号量，在使用时最多只有一个 permit 可用，可以当作独占锁。这就是二进制信号，因为它只有两种状态，可用和不可用。
> 这在一些特殊情况下很有用，比如说死锁恢复。因为 release 可以由别的线程调用。

> **公平/非公平**
> Semaphore 类的其中一个构造方法接收一个公平性参数，用于获取公平/非公平信号量，非公平允许竞争，公平保证 FIFO。默认情况下，总是返回非公平的实例。同样的，tryXXX 方法不遵循公平规定。
> 一般来说，保证公平性能够确保线程不会长时间饥饿；使用非公平性能获得更高的吞吐量。

> **内存一致性**
> 一个线程调用 release 方法 happen-before 另一个线程调用 acquire 方法

```java
public class Semaphore implements java.io.Serializable
```

**Usage**
> Before obtaining an item each thread must acquire a permit from the semaphore, guaranteeing that an item is available for use. When the thread has finished with the item it is returned back to the pool and a permit is returned to the semaphore, allowing another thread to acquire that item.

```java
class Pool {    
    private static final int MAX_AVAILABLE = 100;    
    private final Semaphore available = new Semaphore(MAX_AVAILABLE, true);      
    public Object getItem() throws InterruptedException {      
        available.acquire();      
        return getNextAvailableItem();    
    }      
    public void putItem(Object x) {      
        if (markAsUnused(x))        
        available.release();    
    }
}    
```

<br>

#### 内部类

##### Sync

```java
abstract static class Sync extends AbstractQueuedSynchronizer
```

**内部属性/方法**

```java
// 实际上 permit 就是 state
Sync(int permits) { setState(permits); }

final int getPermits() { return getState(); }
// 获取非公平共享信号量
final int nonfairTryAcquireShared(int acquires) {
    for (;;) {
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
// 释放共享信号量
protected final boolean tryReleaseShared(int releases) {}

```

<br>

##### FairSync

```java
static final class FairSync extends Sync
```

```java
protected int tryAcquireShared(int acquires) {
    for (;;) {
        if (hasQueuedPredecessors()) // 检查是否有队列在等待获取信号量
            return -1;
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```

<br>

##### NonfairSync

```java
static final class NonfairSync extends Sync
```

```java
//  获取共享信号量
protected int tryAcquireShared(int acquires) {
    return nonfairTryAcquireShared(acquires);
}
```

#### 内部方法
```java
// 获取信号量，permit - 1，如果不可用则阻塞知道获取成功或当前线程中断
public void acquire() throws InterruptedException {
    // Acquires in shared mode, aborting if interrupted.
    sync.acquireSharedInterruptibly(1);
}
// 不遵守公平规则获取 permit，如果不可用则阻塞知道获取成功或当前线程中断
public boolean tryAcquire() {
    return sync.nonfairTryAcquireShared(1) >= 0;
}
// 释放信号量，permit + 1
public void release() {
    sync.releaseShared(1);
}
```

<br>

### CountDownLatch

> 一个同步辅助工具，允许一个或多个线程等待，直到其他线程中正在执行的一组操作完成。可以使用一个给定的计数来初始化 CountDownLatch。线程调用 await 方法后会进入阻塞，直到计数器由于 countDown 方法减为 0，处于等待的线程被唤醒执行。CyclicBarrier 的作用与 CountDownLatch 相反

> CountDownLatch 是一个多功能的同步工具，可用于多种用途。一个初始化为 1 的 CountDownLatch 可以作为一个简单的开/关锁：所有调用 await 的线程都在等待，直到 countDown 调用到计数为 0；一个初始化为 N 的 CountDownLatch 可以用来让一个线程等待，直到 N 个线程执行完成，或者某个操作已经完成了 N 次。

```java
public class CountDownLatch
```

**内部/类/属性/方法**
```java
private static final class Sync extends AbstractQueuedSynchronizer {
    Sync(int count) { setState(count); }
    int getCount() { return getState(); }
    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }
    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
}

public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
public void countDown() { sync.releaseShared(1); }
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1); 
}
```

**Usage**

<br>

### CyclicBarrier

> 一个同步辅助工具，允许一组线程互相等待，以达到指定的触发值。效果和 CountDownLatch 相反。屏障被称为循环的，因为它在等待的线程被释放后可以被重新使用，使用 reset 方法会重新生成一个 Generation，即可重新利用 CyclicBarrier。
> CyclicBarrier 支持一个可选的 Runnable 命令，该命令在最后一个线程到达之后，在释放任何线程之前，运行一次

> **内存一致性**
> 一个线程调用 await 操作 happen-before 屏障操作；而屏障操作又 happen-before 其他线程成功调用 await 后执行的操作

```java
public class CyclicBarrier
```

```java
// barrier 的每次使用都被表示为一个 Generation 实例，barrier 改变时 Generation 实例也会跟着变化。可能会有很多 Generation 实例和使用 barrier 的线程相关联。
private static class Generation { boolean broken = false; }

// 指定触发值，创建 CyclicBarrier 实例
public CyclicBarrier(int parties) {
    this(parties, null);
}

// 指定触发值 parties，并指定达到触发值后运行的操作 barrierAction，创建 CyclicBarrier 实例
public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierAction;
}

// Resets the barrier to its initial state.
// If any parties are currently waiting at the barrier, they will return with a BrokenBarrierException.
public void reset() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        breakBarrier();   // break the current generation
        nextGeneration(); // start a new generation
    } finally {
        lock.unlock();
    }
}
```

**Usage**

```java
public class CyclicBarrierTest {
    public static void main(String[] args) {
        CyclicBarrier barrier = new CyclicBarrier(7, () -> {
            System.out.println("All works done");
        });
        for (int i = 0; i < 7; i++) {
            new Thread(() -> {
                System.out.println(Thread.currentThread().getName() + " => DONE");
                try {
                    barrier.await(); // 当前线程工作完成，进入等待
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                } catch (BrokenBarrierException e) {
                    throw new RuntimeException(e);
                }
            }, "th-" + i).start();
        }
    }
}
```



<br>

### Phaser

> 分段锁，一个可重用的同步屏障，在功能上与 CyclicBarrier 和 CountDownLatch 相似，但更灵活。它允许多个线程协调执行，并在到达同步点时同步它们的进度。
> 
> Phaser 中最基本的概念是“阶段（phase）”，它代表了一个同步点。Phaser 可以有任意数量的阶段，每个阶段都有一个唯一的标识符。

> Phaser 的主要操作是 arrive 和 awaitAdvance
> * arrive 方法通知 Phaser 当前线程已经到达当前阶段的同步点，它会阻塞等待所有其他线程到达。一旦所有线程都到达，Phaser 会将阶段递增，并唤醒所有等待的线程。
> 
> * awaitAdvance 方法允许线程等待 Phaser 进入下一个阶段。该方法返回当前阶段的标识符，如果 Phaser 已经进入了下一个阶段，则该方法将一直阻塞，直到进入下一个阶段为止。
> 
> Phaser 还提供了其他一些方法，例如 register，允许线程将自己注册到 Phaser 中，以便在 Phaser 进入下一个阶段时被唤醒。Phaser 还提供了一些高级功能，例如可中断的等待和超时等待。
> 
> Phaser 是一种非常灵活和可扩展的同步器，可以用于许多不同的并发场景，例如分段并发执行，分治算法，以及多线程任务的分组等。

> **注册**
> 线程任务可以在任何时候将自己注册到 Phaser。可以使用 register/bulkRegister 方法注册或构造器注册；也可以在任何到达的时候使用  arriveAndDeregister 取消注册。
> 注册和注销只影响内部 count 计数，外部线程任务不能查询它们是否被注册。

> **同步机制**
> 和 CyclicBarrier 一样，Phaser 可以被反复 await，arriveAndAwaitAdvance 方法的效果类似于 CyclicBarrier.await。
> 每一段 phaser 都有一个与之相对应的 phase number，phase number 从 0 开始。当所有的注册的线程任务都达到 phase 时 phase number + 1，在达到 Integer.MAX_VALUE 后重置为 0。
>
> phase number 可以独立控制**到达某一阶段 phaser**和**等待其他线程**这两个动作
> * Arrival
>   arrive 和 arriveAndDeregister 方法负责记到达状态，这两个方法不会阻塞，会返回相关的 arrival phase number 用来确认到达状态。当所有注册的线程任务都到达了同一个 phase，一个可选的 action 就会执行，phase 也会向前推进。可选操作通过重写 onAdvance 方法实现，通常可以用来控制终止状态，此方法类似于 CyclicBarrier 提供的 barrierAction，但比它更灵活。
> * Waiting
>   awaitAdvance 方法需要一个表示 arrival phase number 的参数，并且在 phaser 前进到与给定 phase 不同的 phase 时返回。和 CyclicBarrier 不同，即使等待线程已经被中断，awaitAdvance 方法也会一直等待。中断状态和超时时间同样可用，但是当任务等待中断或超时后未改变 phaser 的状态时会报异常。如果有必要，在方法 forceTermination 之后可以执行这些异常的相关的 handler 进行恢复操作。Phaser 也可能被 ForkJoinPool 中的任务使用，这样在其他任务阻塞等待一个 phase 时可以保证足够的并行度来执行任务。

> **终止机制**
>可以使用 isTerminated 方法检查 Phaser 是否处于终止状态。一旦终止，所有的 arrive 方法立即返回负数。在终止时注册是无效的。当调用 onAdvance 返回 true 时 Termination 被触发。当 deregistration 操作使已注册的线程任务变为 0 时，onAdvance 的默认实现就会返回 true。也可以重写 onAdvance 方法来定义终止动作。forceTermination 方法也可以释放等待线程并且允许它们终止。

> **分层结构**
>Phaser 支持分层结构(树状构造)来减少竞争。注册了大量线程任务的 Phaser 可能会因为同步竞争消耗很高的成本，因此可以设置一些子 Phaser 来共享一个通用的 parent。即使每个操作消耗了更多的开销，也能提高整体吞吐量。
>在一棵分层 Phaser 树中，子 Phaser 与其父 Phaser 的注册和取消注册由父结点自动管理。 每当子 Phaser 注册数量不为 0，该 Phaser 就会向其父 Phaser 进行注册。 每当 arriveAndDeregister 调用后发现注册数量变为零时，该子 Phaser 将从其父 Phaser 中取消注册。

> **状态监控**
> 虽然同步方法只能由注册方调用，但 Phaser 的当前状态可由任何调用者监控。在任何给定的时刻，总共有 getRegisteredParties 个参与者，其中 getArrivedParties 个已经到达了当前阶段(getPhase)。当剩余的参与者（getUnarrivedParties）到达时，该阶段向前推进。
> 
> 这些方法返回的值可能会反映出瞬态状态，因此通常不适合用于同步控制。toString 方法能以非正式监测的形式返回这些状态查询快照。

**内部类/属性/方法**
```java
static final class QNode implements ForkJoinPool.ManagedBlocker {}

private volatile long state;

// 注册一个新的 party
public int register()
// 批量注册
public int bulkRegister(int parties)
// 使当前线程到达 phaser，不等待其他任务到达。返回 arrival phase number
public int arrive() 
// 使当前线程到达 phaser 并撤销注册，返回 arrival phase number
public int arriveAndDeregister()
/**
 * 使当前线程到达 phaser 并等待其他任务到达，等价于 awaitAdvance(arrive())。
 * 如果需要等待中断或超时，可以使用 awaitAdvance 方法完成一个类似的构造。
 * 如果需要在到达后取消注册，可以使用 awaitAdvance(arriveAndDeregister())。
 */
public int arriveAndAwaitAdvance()
//等待给定 phase 数，返回下一个 arrival phase number
public int awaitAdvance(int phase)
//阻塞等待，直到 phase 前进到下一代，返回下一代的 phase number
public int awaitAdvance(int phase) 
//响应中断版 awaitAdvance
public int awaitAdvanceInterruptibly(int phase) throws InterruptedException
public int awaitAdvanceInterruptibly(int phase, long timeout, TimeUnit unit)
    throws InterruptedException, TimeoutException
//使当前 phaser 进入终止状态，已注册的 parties 不受影响，如果是分层结构，则终止所有 phaser
public void forceTermination()
```
<br>

**Usage**
```java
void runTasks(List<Runnable> tasks) {    
	 final Phaser phaser = new Phaser(1); 
	 // create and start threads    
	 for (final Runnable task : tasks) {      
		 phaser.register();      
		 new Thread() {        
			 public void run() {          
			     phaser.arriveAndAwaitAdvance(); // await all creation
					task.run();        
				}      
			}.start();    
		}      
		// allow threads to start and deregister self
		phaser.arriveAndDeregister();  
}
```

> ...

<br>

## 并发集合

### ConcurrentHashMap

### CopyOnWriteArrayList

### ConcurrentSkipListMap

### ConcurrentSkipListSet


<br>

## Atomic

### AtomicInteger

### AtomicLong

### AtomicReference

### AtomicReferenceArray

### AtomicStampedReference

### DoubleAdder

### LongAdder