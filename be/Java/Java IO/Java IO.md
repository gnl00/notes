# Java IO

## 前言

> Java 中的 IO 操作，包括 BIO（*Blocking IO*）和 NIO（*Non-Blocking IO*）。在处理大量数据时，BIO 往往无法满足性能要求，推荐使用 NIO。
>
> 在实际应用中，BIO 适用于连接数比较少的情况下，如数据库连接、文件上传等；而 NIO 适用于连接数较多且连接时间较短的情况下，如聊天服务器、在线游戏等。

|          | BIO           | NIO               |
| -------- | ------------- | ----------------- |
| 读写方式 | 字符流/字节流 | Channel           |
| 缓冲     | 读写缓冲      | 读写缓冲          |
| 连接     | 一连接一线程  | Selector 多路复用 |



> **IO 多路复用技术**
>
> 通常使用的 IO 模型包括：阻塞 IO、非阻塞 IO、IO 多路复用和异步 IO。
>
> * 在阻塞 IO 模型中，线程会一直等待 IO 操作完成
> * 非阻塞 IO 模型中，线程不会等待 IO 操作完成，而是通过轮询的方式不断查询 IO 操作状态
> * IO 多路复用模型通过统一的系统调用，将多个 IO 事件注册到同一个地方（Channel 注册到 Selector），当有 IO 事件就绪时，就会通知线程进行处理，在同一个线程中处理多个 IO 操作。
> * 异步 IO 技术是指 IO 操作完成后会通过操作系统发送一个信号或回调函数来通知应用程序，无需通过轮询的方式来等待操作结果
>
> <br>
>
> **IO 多路复用技术使用到的方法**
>
> *下面这些方法都是操作系统提供的`系统调用方法`*
>
> * select
>
>   select 函数是最早的多路复用函数之一，可监视的文件描述符（fd_set）数量有限（BitMap 结构，默认长度为 1024），实现方式是遍历整个 fd_set 集合，时间复杂度为 O(N)。select 每次调用都会把 fd_set 集合从用户态拷贝到内核态，由内核来遍历 fd_set，所以性能较低。此外，select 函数的实现方式要求用户传递 fd_set 集合，所以它存在一定的安全漏洞。
>
> * poll
>
>   poll 函数可监视的文件描述符数量不受限制。poll 函数通过链表来保存文件描述符，时间复杂度为 O(N)。poll 函数不会每次都把 fd_set 集合从用户态拷贝到内核态，只需要把链表从用户态拷贝到内核态。性能比 select 函数好。
>
> * epoll
>
>   在 select 和 poll 中，用户态每次都需要对整个 fd_set 进行轮询，才能知道有哪个 socket 的 IO 就绪了。
>
>   epoll 函数是 Linux 下的一种 IO 多路复用机制，可监视的文件描述符数量也不受限制。epoll 函数不需要将 fd_set 集合在用户和内核态中来回拷贝，并且对就绪 IO 进行了标记，不再需要对整个 fd_set 进行轮询操作，效率比 select 和 poll 高。而且 epoll 内部使用红黑树来管理 fd_set，减少遍历 fd_set 的时间复杂度，时间复杂度为 O(log N) 。
>
> <br>
>
> fd_set 是一种数据结构，是一个集合，用于存储被监控的文件描述符（*file descriptor*）。常被用于 IO 操作，比如 IO 多路复用技术中 select/poll/epoll 都使用到了它。

> Java 的 NIO 实现早期版本使用的是 select，从 JDK 1.5 开始采用 epoll。

> 网络 IO 模型就是围绕着使用尽可能少的线程处理更多的请求这一方向演变的。

<br>

**零拷贝**

> 零拷贝（Zero Copy）是指数据在不需要经过用户态和内核态之间的数据拷贝，直接在内核空间中进行数据传输的技术。主要是为了解决数据传输过程中频繁拷贝所带来的性能问题。
>
> 在传统的 IO 操作中，数据需要先从磁盘或网络中读取到内核空间，再从内核空间拷贝到用户空间，最后再通过网络传输给接收方。这种传输方式会导致数据在用户空间和内核空间之间频繁拷贝，造成额外的 CPU 开销和内存带宽消耗，影响系统的性能。而零拷贝技术则可以避免这种数据拷贝，将数据在内核空间中直接进行传输。
>
> 零拷贝技术的出现，主要是为了提升数据传输的性能。在数据传输过程中，零拷贝可以减少拷贝操作，从而减少了 CPU 的消耗和内存带宽的使用。此外，零拷贝还可以降低应用程序的内存占用，提高系统的吞吐量。

> 在 Java 中，零拷贝技术可以通过 NIO 中的 FileChannel 类实现。FileChannel 提供了 transferTo() 和 transferFrom() 方法，可以直接将数据在内核空间中进行传输，避免了数据在用户空间和内核空间之间的拷贝。同时，Java 也提供了 MappedByteBuffer 类，可以将文件直接映射到内存中，从而避免了文件在内存和磁盘之间的拷贝。



<br>

## BIO

> BIO 是传统的 IO 操作，它是一种同步阻塞的IO模式，即在执行输入/输出操作时，线程会被阻塞，直到操作完成。这意味着当一个线程在执行 IO 操作时，其他线程将被阻塞，直到该操作完成。
>
> 在 Java 中，BIO 是通过 InputStream 和 OutputStream 类来实现的
>
> ```java
> try (InputStream in = new FileInputStream("file.txt")) {
>     int b;
>     while ((b = in.read()) != -1) {
>         // 处理读取到的数据
>     }
> } catch (IOException e) {
>     // 处理异常
> }
> ```
>
> 在上面的代码中，InputStream 类的 read 方法将被阻塞，直到它从文件中读取到数据。由于该操作是同步的，因此该线程将一直处于阻塞状态，直到该操作完成。如果有多个线程执行类似的操作，则这些线程将相互阻塞，导致应用程序的性能降低。

> BIO 阻塞式 IO 模型，每次连接都需要单独开启线程来处理读写操作，当并发连接数比较大时，会创建大量的线程，占用大量的系统资源，容易导致系统崩溃。

> BIO 模型中，ServerSocket 负责绑定 IP 地址，启动监听端口；Socket 负责发起连接操作。连接成功后，双方通过 InputStream 和 OutputStream 进行同步阻塞式数据传输。

…

**适用场景**

* 连接数少
* 并发度低

---

<br>

## NIO

> NIO 是基于多路复用器的非阻塞式 IO，在执行输入/输出操作时，线程不会被阻塞，可以继续执行其他操作。这使得一个线程可以同时处理多个输入/输出操作。

> NIO 中的核心组件有三个：Channel（通道）、Buffer（缓冲区）和 Selector（选择器）。Channel 负责传输数据，Buffer 负责存储数据，Selector 负责监听通道上的事件，并将事件分发给对应的线程进行处理。
>
> 它采用一个线程处理多个连接，即在一个线程中可以监听多个 Channel，这些 Channel 都绑定在同一个 Selector 上，当其中的一个 Channel 有事件时（如可读、可写等），Selector 就会通知对应的线程进行处理。

> 在 Java 中，NIO 是通过 Channel、Buffer、Selector 类来实现的
>
> ```java
> try (RandomAccessFile file = new RandomAccessFile("file.txt", "r")) {
>     FileChannel channel = file.getChannel();
>     ByteBuffer buffer = ByteBuffer.allocate(1024);
>     int bytesRead = channel.read(buffer);
>     while (bytesRead != -1) {
>         buffer.flip();
>         while (buffer.hasRemaining()) {
>             // 处理读取到的数据
>             System.out.print((char) buffer.get());
>         }
>         buffer.clear();
>         bytesRead = channel.read(buffer);
>     }
> } catch (IOException e) {
>     // 处理异常
> }
> ```
>
> 在上面的代码中，FileChannel 的 read 方法将不会阻塞线程。使用 Buffer 类可以缓存数据，在处理数据时更加高效。NIO 允许一个线程通过一个 Channel 处理多个 IO 操作。

> NIO 采用 **Reactor 模式**，一个线程可以处理多个请求，当请求来临时，只需进行事件注册，而不是像 BIO 一样开启一个线程处理一个请求。

> Reactor 模式是一种基于事件驱动、非阻塞式 IO 的编程模式，相对于传统的阻塞式 IO，它的主要优点有
>
> 1. 高并发性：Reactor 采用异步非阻塞 IO，允许多个请求同时被处理，能够提高系统的并发能力。
> 2. 高吞吐量：异步非阻塞 IO，允许一个线程处理多个请求，能够提高系统的吞吐量。
> 3. 低延迟：由于异步非阻塞 IO，一个线程可以处理多个请求，避免了线程切换和上下文切换的开销，从而降低延迟。

> 相比于传统的编程模式，Reactor 模式具有以下特点：
>
> 1. 事件驱动：Reactor 模式是基于事件驱动的，当有事件发生时，会通知相关的事件处理器进行处理。
> 2. 非阻塞式 IO：Reactor 模式采用非阻塞式 IO，当 IO 操作无法立即完成时，线程可以继续进行其他操作，避免了线程的阻塞等待。
> 3. 多路复用：Reactor 模式采用多路复用技术，允许一个线程处理多个请求。
>
> <br>
>
> Reactor 模式适用于高并发请求的场景，例如高性能服务器、大型分布式系统等。在 Reactor 模式中，可以通过线程池来处理请求，进一步提高系统的并发性能。

> NIO 模型中，ServerSocketChannel 负责绑定 IP 地址，启动监听端口；SocketChannel 负责发起连接操作。连接成功后，双方通过 Channel 进行异步非阻塞式数据传输，可以将读写事件的状态保存在 Buffer 中，当缓冲区满时，会触发读写事件。



<br>

### Buffer

> 一个用于存放特定原始类型数据的容器，还存储数据容量，数据地址，数据大小限制等。定义于 `java.nio` 包内。
>
> **只读 Buffer**
>
> 每一个 Buffer 都是可读的，但不是每个 Buffer 都是可写的。对只读缓冲区进行写入操作会报错 ReadOnlyBufferException。
>
> **线程安全**
>
> 缓冲区对于多个并发线程的使用是不安全的。如果一个缓冲区要被一个以上的线程使用，对缓冲区的访问应该通过适当的同步来控制。

```java
public abstract class Buffer
```



<br>

### Channel

> NIO 数据传输的纽带（通道），定义于 `java.nio.channels` 包内。

```java
public interface Channel extends Closeable
```

```java
// Channel 只能是开启或者关闭状态
public boolean isOpen();
public void close() throws IOException;
```



<br>

### Selector

> 一个 SelectableChannel 对象的多路复用器

```java
public abstract class Selector implements Closeable
```

```java
public static Selector open() throws IOException {
    return SelectorProvider.provider().openSelector();
}
public abstract boolean isOpen();
public abstract SelectorProvider provider();
public abstract Set<SelectionKey> keys(); // Returns this selector's key set.
public abstract Set<SelectionKey> selectedKeys(); // Returns this selector's selected-key set
public abstract int selectNow() throws IOException; // Selects a set of keys whose corresponding channels are ready for I/O operations.
public abstract int select() throws IOException; // Selects a set of keys whose corresponding channels are ready for I/O operations
public abstract Selector wakeup(); // Causes the first selection operation that has not yet returned to return immediately.
public abstract void close() throws IOException; // Closes this selector.
```

> Selector#open 方法会使用系统默认的 SelectorProvider 创建一个 Selector；也可以使用自定义的 SelectorProvider 来创建 Selector。Selector 一直是 open 的，直到它被关闭。

> 将 SelectableChannel 注册到 Selector 上是通过 SelectionKey 来表示的，Selector 维护 3 个 SelectionKey 集合。
>
> * *Key Set*，注册到 Selector 上的 SelectionKey 集合。
> * *Selected-Key Set*，就绪事件的 SelectionKey 集合，用于获取已经发生了就绪事件的通道，可以从中获取通道的就绪事件类型（例如读就绪、写就绪等），然后进行相应的操作。
> * *Cancelled-Key Set*，已取消的 SelectionKey 集合，在 Selector 的操作过程中，可能会取消某些 SelectionKey，这些已取消的 SelectionKey 会被添加到当前集合中。用于清理无效的 SelectionKey 对象，释放资源并进行垃圾回收。
>
> 在 Selector 刚创建的时候以上 3 个集合都是空的。对于应用程序来说，通常只需要关注 *Selected-Key Set*，通过遍历该集合来处理已就绪的通道。

> **Selection 操作**
>
> Selection 操作是用于检查注册在 Selector 上的通道是否有就绪事件（例如读就绪、写就绪等），并将就绪的通道的 SelectionKey 添加到已选择键集合中。
>
> Selection 操作主要涉及以下三个步骤：
>
> 1. **注册通道**：首先，将一个或多个通道注册到 Selector 上，通过调用通道的 SelectableChannel#register 方法，并传入 Selector 以及感兴趣的事件类型（例如 OP_READ、OP_WRITE 等）。注册成功后，通道与 SelectionKey 建立关联，并将 SelectionKey 添加到 Selector 的键集合中。
> 2. **选择通道**：然后，调用 Selector#select 方法进行通道的选择。在这个步骤中，Selector 会阻塞，直到至少一个注册的通道就绪或超时时间到达。select() 方法返回的是就绪事件的数量。
> 3. **处理就绪通道**：一旦 select() 方法返回大于 0 的就绪事件数量，就表示有通道就绪。此时，可以通过调用 Selector#selectedKeys 方法获取已选择键集合，遍历其中的 SelectionKey，获取每个就绪通道的就绪事件类型，并进行相应的处理。处理完毕后，需要手动从已选择键集合中移除已处理的 SelectionKey，以确保下次循环时不会再次处理已就绪的通道。

> **并发支持**
>
> Selector 本身是线程安全的，但是它维护的 SelectionKey 集合却不是。Selection 操作会在 Selector 本身，以及 *Key Set* 和 *Selected-Key Set* 上同时进行。在对 SelectionKey 集合上进行操作的时候需要注意维护相应的同步机制，避免出现线程安全问题。

```java
// SelectorImpl

private final Set<SelectionKey> selectedKeys;
private final Set<SelectionKey> publicSelectedKeys; // Removal allowed, but not addition

protected SelectorImpl(SelectorProvider sp) {
    // ...
  	// selectedKeys 实际上就是 HashSet，多线程操作下线程不安全
    selectedKeys = new HashSet<>();
    // ...
    publicSelectedKeys = Util.ungrowableSet(selectedKeys);
}

@Override
public final Set<SelectionKey> selectedKeys() {
    ensureOpen();
    return publicSelectedKeys;
}
```

…

<br>

### 事件驱动

在 BIO 中，IO 读写需要经过一段时间的阻塞，等待获取到数据才返回。与 BIO 的关心返回数据不同，NIO 更关心的是是否有自己感兴趣的事件（有可读/写事件），有则进行事件操作（读/写），没有则返回 0，不会阻塞线程。

NIO 的主要事件有：读就绪、写就绪、有新连接到来。因此，使用 NIO 流程应该是：

1、注册感兴趣的事件（新连接/读/写），Java 中的表现为将 Channel 注册到 Selector 中；

2、等待事件到来。Selector 阻塞等待新事件，接收到新事件后保存到对应的集合（*Selected-Key Set*）。Java 中通过 Selector#select 可以感知到是否有新事件；

3、最后通过 Selector#selectedKeys 获取可以处理的事件集合并处理事件。



<br>

### 基础使用

创建一个基于 NIO 的服务器

```java
public class SimpleServer {
    public static void main(String[] args) throws IOException {
        // server channel
        ServerSocketChannel socketChannel = ServerSocketChannel.open();
        socketChannel.configureBlocking(false); // set non-block
        Selector selector = Selector.open(); // open Selector
        SelectionKey selectionKey = socketChannel.register(selector, 0, socketChannel); // register channel
        // 设置感兴趣的事件，服务器通道只能注册 SelectionKey.OP_ACCEPT 事件
        selectionKey.interestOps(SelectionKey.OP_ACCEPT);
        socketChannel.bind(new InetSocketAddress(8888)); // server channel 绑定端口

        while (true) { // selector 开始轮询
            // select 方法会阻塞，直到有事件发生
            if (selector.select() == 0) { // 无事件发生
                continue;
            }
            Set<SelectionKey> keys = selector.selectedKeys();
            Iterator<SelectionKey> iterator = keys.iterator();
            while (iterator.hasNext()) {
                SelectionKey key = iterator.next();
                // remove after get a key
                // 将已经处理事件从 selectedKeys 中移除，如果不移除，会一直在 selectedKeys 集合中，下一次循环会重复处理该事件
                iterator.remove();

                // 如果轮询到建立连接事件，注册到 selector 中
                if (key.isAcceptable()) {
                    ServerSocketChannel serverChannel = (ServerSocketChannel) key.channel(); // get server channel

                    // get client channel
                    SocketChannel clientChannel = serverChannel.accept();
                    clientChannel.configureBlocking(false);
                  
                    SelectionKey clientSelectionKey = clientChannel.register(selector, 0, clientChannel); // register client channel
                    clientSelectionKey.interestOps(SelectionKey.OP_READ);
                    System.out.println("[acceptable] client connected");
                    clientChannel.write(ByteBuffer.wrap("connect success".getBytes())); // 客户端连接成功
                    System.out.println("[acceptable] server responded");
                }

                if (key.isReadable()) { // 如果轮询到可读事件
                    // get client channel
                    SocketChannel clientChannel = (SocketChannel) key.channel();
                    // set client data to buffer
                    ByteBuffer buffer = ByteBuffer.allocate(1024);
                    // write into buffer
                    int len = clientChannel.read(buffer);
                    System.out.println("[readable] server receive byte length: " + len);
                    if (len > 0) {
                        buffer.flip(); // 将缓存区从写状态切换为读状态（实际上这个方法是读写模式互切换）
                        System.out.println("[readable] server received: " + Charset.defaultCharset().decode(buffer).toString());
                    } else {
                        clientChannel.close();
                        break;
                    }
                }
            }
        }
    }
}
```



创建基于 NIO 的客户端

```java
public class SimpleClient {
    public static void main(String[] args) throws IOException {
        SocketChannel clientChannel = SocketChannel.open();
        clientChannel.configureBlocking(false);
        // get selector
        try (Selector selector = Selector.open()) {
            clientChannel.connect(new InetSocketAddress(8888)); // 连接服务器
            // 将客服端注册到 selector 上
            SelectionKey selectionKey = clientChannel.register(selector, 0);
            // 设置感兴趣的事件
            selectionKey.interestOps(SelectionKey.OP_CONNECT);
            
            while (true) { // 开始轮询事件
                if (selector.select() == 0) { // 阻塞，直到有事件
                    continue;
                }
                Set<SelectionKey> selectionKeys = selector.selectedKeys();
                Iterator<SelectionKey> iterator = selectionKeys.iterator();
                while (iterator.hasNext()) {
                    SelectionKey key = iterator.next();
                    iterator.remove(); // remove after get

                    if (key.isConnectable()) { // 如果是连接成功事件
                        if (clientChannel.finishConnect()) {
                            // 注册读事件
                            clientChannel.register(selector, SelectionKey.OP_READ);
                            System.out.println("[connectable] read event register success");
                            clientChannel.write(ByteBuffer.wrap("This is message from client".getBytes()));
                        }
                    }

                    if (key.isReadable()) { // 如果是可读事件
                        SocketChannel channel = (SocketChannel) key.channel();
                        // allocate buffer
                        ByteBuffer buffer = ByteBuffer.allocate(1024);
                        int len = channel.read(buffer);
                        System.out.println("[readable] client receive byte length: " + len);

                        byte[] readByte = new byte[len];
                        buffer.flip();
                        buffer.get(readByte);
                        System.out.println("[readable] client received: " + new String(readByte));
                    }
                }
            }

        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```





<br>

## AIO

> AIO（Asynchronous IO）是 Java NIO 的扩展，也称为 NIO.2，它是异步非阻塞的 IO 操作，相对于传统的 BIO（Blocking IO）和 NIO（Non-blocking IO）都具有更高的效率和可靠性。

> AIO 采用事件驱动机制，应用程序通过操作系统注册 IO 操作，并在操作系统完成操作后，操作系统会通知应用程序进行后续操作，这样就可以避免阻塞和轮询的问题。相比较 BIO 和 NIO，AIO 更适合处理大量的连接请求和数据量大的情况，具有更高的吞吐量和并发能力，同时减少了 CPU 和内存的开销。

> * BIO（Blocking IO）是传统的阻塞式 IO 操作，每个连接都需要独立的线程进行阻塞式的等待读写操作。
>
> * NIO（Non-blocking IO）则是基于事件驱动模型的 IO 操作，通过 Selector 机制实现单线程管理多个连接，并监听各个连接的状态，处理连接状态变化的事件。相比较 BIO，NIO 可以减少线程的开销，但是依然需要遍历连接进行轮询，效率并不是特别高。
>
> * 而 AIO 利用操作系统异步 IO 支持，可以在连接数据准备好后异步通知应用程序进行后续操作，大大提高了效率和可靠性。
>
> 在 Java 中，AIO 的实现主要是基于 Java NIO 中的 AsynchronousServerSocketChannel 和 AsynchronousSocketChannel，通过注册 IO 操作和实现 CompletionHandler 接口来实现异步 IO。





<br>

## 参考

https://tech.meituan.com/2016/11/04/nio.html