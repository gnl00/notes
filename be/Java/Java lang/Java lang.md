# Java lang

> 关于 `java.lang` 包中一些类的笔记



## Runtime

每一个 Java 程序都有一个单实例的 Runtime 对象，它允许正在运行的 Java 程序能够获取到运行环境信息。

一些常用的方法如下：

* `getRuntime()`
* `exec(String command)`
* `availableProcessors()`

<br />

## Process

`ProcessBuilder.start()` 和 `Runtime.exec` 方法会创建一个本地进程，然后返回 Process 子类的一个实例，该实例可以用来控制进行并获取相关信息。

由于创建出来的子进程没有自己的终端或控制台，因此它的所有 IO 操作都是通过：

* `getOutputStream()`
* `getInputStream()`
* `getErrorStream()`

这三个方法将 IO 流重定向到父进程的。

Input 和 Output 都是针对调用 Process 的父进程而言的。如果要获取 Process 子进程输出的数据，需要将 Process 进程的数据 Input 到父进程；如果要将数据输入到 Process 子进程，需要将父进程的数据 Output 到 Process 子进程。



想要获取 Process 子进程的输入数据父进程就可以调用 `getInputStream()` 方法。但是，Process 子进程的输出的错误信息只能从 `getErrorStream()` 获取到，`getInputStream()`  无法获取到错误信息。

所以在获取 Process 子进程输出信息的时候不能只调用 `getInputStream()` 还应该调用 `getErrorStream()` 配合查看输出的信息。





## 参考

### Process

* https://zhuanlan.zhihu.com/p/44957705