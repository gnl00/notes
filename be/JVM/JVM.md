# JVM

## 前言

> **JDK/JRE/JVM**
> 
> * JDK（*Java Development Kit*），Java 开发工具。包含 JRE 和各种工具（javac/java/jdb 等）以及 Java 基础类库 `rt.jar`；
>   
> - JRE（*Java Runtime Environment*），Java 运行时环境，包含 JVM 标准实现以及 Java 核心类库；
>   
> - JVM（*Java Virtual Machine*），JVM 虚拟机，在实际计算机上模拟各种计算机功能。
>   
> 
> 
> **联系**
> 
> - JVM 不能单独完成 `.class` 的运行。解析 `.class` 文件时 JVM 需要调用所需要的类库和工具。类库和工具在 jdk/jre/bin/lib 目录下。
>   
>   可以认为 bin 目录里就是 JVM，lib 则是 JVM 工作时所需要的类库。JVM 和 lib 合起来就成为 JRE，即 `JRE = JVM + lib`；
>   
> - 开发者利用 JDK 调用 Java API 开发 Java 程序，通过 `javac` 将 Java 文件编译成 `.class` 类型的字节码。然后在 JRE 上运行，JVM 负责解析这些字节码，并映射到 CPU 指令集或 OS，进行系统调用。
>   
> 
> 
> **区别**
> 
> - 在 jdk/bin 目录下有 `javac` 等工具，而 jre/bin 目录里没有。`javac` 指令是将 Java 文件编译成 Java 字节码文件的工具，运行环境 JRE 是不需要的；jdk/bin 目录下还有 `jar/javadoc ` 等开发工具，这也说明 JDK 是开发环境，JRE 是运行时环境；
>   
> - 仅 JVM 自身还不能执行 `.class` 文件，还需要 JRE 目录下 lib 类库的支持，尤其是 `rt.jar`。


<br>

## JVM 结构

> JVM 内存结构和内存模型（JMM）是两个不同的概念。

### JVM 内存结构体系

![JVM内存结构](./assets/JVM内存结构.jpg)

<br>

### 类加载子系统

#### 类加载过程

类的加载分为加载、连接和初始化三个阶段。

##### 加载

加载阶段负责加载 Java 字节码文件，从 `.class` 到 `Class`。流程如下：

1、通过类的*全限定类名*获取定义此类的二进制字节流；

2、将这个字节流所代表的*静态存储结构*转化为方法区的*运行时数据结构*；

3、在内存中生成一个代表此类的 `java.lang.Class` 对象，并将其引用存放在方法区，作为此类数据的访问入口。

> 关于字节码这里有一篇[文章](https://tech.meituan.com/2019/09/05/java-bytecode-enhancement.html)。

<br>

##### 连接

连接阶段分为验证、准备和解析三个步骤：

1、验证（Verify），目的在于确保字节码文件中包含的信息符合要求并且是安全的，*保证被加载类的正确性*。主要包括四种验证：文件格式验证，元数据验证，字节码验证，符号引用验证。

使用 2 进制编辑器打开 .class 文件可以看到以下内容：

```
CA FE BA BE 00 00 00 34 00 27 0A 00 06 00 18 09 00 19 00 1A 07 00 1B 08 00 1C 0A 00 
```

所有合法的 Java 字节码文件都是以 `CA FE BA BE` 开头的，类加载子系统的验证阶段也会验证这一字段。

2、准备（Prepare）为类静态变量分配内存，并**初始化为默认值**；

> *注意*：当前只是初始化为默认值，并没有进行赋值操作。此时会把编译时确定的常量值存储到对应的常量池中，在后续的初始化阶段，静态变量才会被赋值为定义的值（通过静态代码块/静态方法）。
> 
这也是为什么 `final static` 修饰的变量必须要在定义时初始化，最晚只能在构造方法中初始化，且只能被赋值一次，因为它们在类加载过程中就已经被确定了其值，而无法在后续步骤中改变。

3、解析（Resolve）

将常量池中的*符号引用*转换为*直接引用*。符号引用就是用一组符号来描述所引用的目标，直接引用就是直接指向目标地址的指针、相对偏移量或一个间接定位到目标的语句。

> 使用 `javap -v` 反编译 .class 字节码时会得到一系列常量池信息：
>
> ```
> Constant pool:
>    #1 = Methodref          #6.#24         // java/lang/Object."<init>":()V
>    #2 = Fieldref           #25.#26        // java/lang/System.out:Ljava/io/PrintStream;
>    #3 = Class              #27            // jvm/ClassLoadTest
>    #4 = String             #28            // hello, JVM
> ```
>
> 解析动作主要针对类或接口、字段、类方法、接口方法、方法类型等。对应常量池中的 `Class`、`Fieldref`、`Methodref` 等。

<br>

##### 初始化

先来看一段简单的代码：

```java
public class ClassLoadTest {
    public static final String JVM = "JVM";
    public static void main(String[] args) {
        System.out.println("hello, " + JVM);
    }
}
```

下面是其字节码反编译后的内容：

```java
// class version 52.0 (52)
// access flags 0x21
public class jvm/ClassLoadTest {
  // access flags 0x9
  public static Ljava/lang/String; STR

  // access flags 0x19
  public final static Ljava/lang/String; JVM = "JVM"

  // access flags 0x1
  public <init>()V
   L0
    LINENUMBER 3 L0
    ALOAD 0
    INVOKESPECIAL java/lang/Object.<init> ()V
    RETURN
   L1
    LOCALVARIABLE this Ljvm/ClassLoadTest; L0 L1 0
    MAXSTACK = 1
    MAXLOCALS = 1

  // access flags 0x9
  public static main([Ljava/lang/String;)V
   L0
    LINENUMBER 7 L0
    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
    LDC "hello, JVM"
    INVOKEVIRTUAL java/io/PrintStream.println (Ljava/lang/String;)V
   L1
    LINENUMBER 8 L1
    RETURN
   L2
    LOCALVARIABLE args [Ljava/lang/String; L0 L2 0
    MAXSTACK = 2
    MAXLOCALS = 1

  // access flags 0x8
  static <clinit>()V
   L0
    LINENUMBER 4 L0
    LDC "STR"
    PUTSTATIC jvm/ClassLoadTest.STR : Ljava/lang/String;
    RETURN
    MAXSTACK = 1
    MAXLOCALS = 0
}
```

可以看到，除了 main 方法，其中还多了 `<lint>()` 和 `<clint>()` 两个方法，其中：

* 初始化阶段就是执行类构造方法 `<clint>()` 的过程。此方法不需要定义，是 `javac` 编译器自动收集类中所有类变量的复制动作和静态代码块中的语句合并而来。如果类中没有定义静态变量或静态代码块，不会有此方法。
* 和 `<clint>()` 不一样，`<lint>()` 表示类的构造方法，为类的静态变量赋值，并执行静态代码块。

在类加载过程中，如果类具有父类，会先初始化其父类。JVM 保证子类 `<clint>()` 执行之前，父类的 `<clint>()` 已经执行完毕。如果该类实现了任何接口，则按顺序初始化这些接口。

此外，为了保证多线程环境下类加载的安全性，JVM 必须保证一个类的 `<clint>()` 在多线程下被同步加锁。



<br>

#### 类加载器

> 负责加载字节码文件。类加载器将字节码文件的内容加载到内存中，并将加载的类信息放到方法区中。类加载器只负责字节码文件的加载，至于它是否可以运行则由执行引擎决定。

##### 启动类加载器

*Bootstrap ClassLoader*，由 C/C++ 实现，用来加载 Java 核心类库（`rt.jar/resource.jar` 等），用于加载 JVM 自身需要的类。

启动类加载器<mark>没有父加载器</mark>，是扩展类和系统类加载器的父类加载器，<mark>负责加载扩展类和系统类加载器</mark>。出于安全考虑，启动类加载器只加载包名为 `java/javax/sun` 等开头的类。

<br>

##### 扩展类加载器

*Extension ClassLoader*，由 Java 实现，实现类为 `sun.misc.Launcher$ExtClassLoader`，是 `sun.misc.Launcher` 的匿名内部类。继承 `java.lang.ClassLoader`，除了启动类加载器之外所有的类加载器都继承自此类。

负责从 `java.ext.dirs` 所指定的目录加载类库，或从 jdk/jre/lib/ext 子目录下加载类库。创建自定义的 `jar` 包放在此目录下，也会由扩展类加载器加载。

<br>

##### 系统类/应用程序类加载器

*System/App ClassLoader*，Java 实现，实现类为 `sun.misc.Launcher$AppClassLoader` ，是 `sun.misc.Launcher` 的匿名内部类。同样继承自 `java.lang.ClassLoader`。

负责加载环境变量 `classpath` 下或者系统属性 `java.class.path` 指定路径下的库。是程序中默认的类加载器，一般来说 Java 应用的类都是由它来完成加载。可以通过 `ClassLoader.getSystemClassLoader()` 获取到系统类加载器。

<br>

##### 自定义类加载器

用户可以定制类的加载方式，如果不是很复杂的实现类加载器，可以直接继承自 `java.net.URLClassLoader` 来实现自定义类加载器。需要自定义类加载器的原因：

1. 隔离加载类；
2. 修改类的加载方式
3. 扩展加载源；
4. 防止源码泄露。

<br>

**类加载器继承关系**

<img src="./assets/image-20200430173616539.png" alt="image-20200430173616539" />

<br>

#### 双亲委派机制

JVM 对 `.class` 文件采用的是<mark>按需加载</mark>的方式，当需要使用到该类时才将其字节码文件加载到内存中，生成 Class 对象。

而在加载某个类的字节码文件时，Java 虚拟机采用的是双亲委派机制：把请求层层交给父类加载器处理，父类加载器无法处理时，子类加载器才会尝试去加载，是一种<mark>任务委派模式</mark>。

<img src="./assets/image-20200501160842395.png" alt="image-20200501160842395"  />

<br>

**原理**

1. 类加载器收到了类加载的请求，它不会自己先去加载，而是把这个请求向上委托给父类加载器进行加载；
   
2. 如果父类加载器还存在父类，则进一步向上委托，最终达到最顶层的启动类加载器；
   
3. 如果父类加载器能完成请求的类的加载任务，则由父类加载器进行加载；若是父类加载器无法完成此加载任务（在它的加载路径下没有找到请求加载的 `.class` 文件），子类加载器才会尝试加载。
   

<br>

**优点**

- 采用双亲委派机制的可以**避免类的重复加载**。比如加载位于 `rt.jar` 包中的 `java.lang.Object` 类时，不管是请求哪个类加载器，都会层层向上委托到启动类加载器，使用启动类加载器进行加载。这样就保证使用不同的类加载器最终得到的都是同一个 Class 实例化出来的 `Object` 对象；
  
- <mark>防止子类加载器加载已经存在的核心类</mark>；
  
- 双亲委派机制<mark>保证 JVM 沙箱安全</mark>。
  

<br>

#### 沙箱安全机制

自定义一个名为 String 类，定义并执行 main 方法，在 JVM 加载自定义的 String 类时会率先使用启动类加载器进行加载，而启动类加载器在加载过程中会先加载 JDK 自带的 String 类（`rt.jar` 包中的 `java.lang.String.class`）。

```java
// 自定义 java.lang.String
public class String {
    public static void main(String[] args) {
      	// 报错：在类 java.lang.String 中找不到 main 方法
        System.out.println("this is my java.lang.String");
    }
}
```

运行自定义的 String 类时会报错 `在类 java.lang.String 中找不到 main 方法`，原因是启动类加载器加载的是 `java.lang.String` 类而不是自定义的 String 类。这样可以保证对 Java 核心代码的保护，这就是<mark>沙箱安全机制</mark>。

<br>

#### 类加载细节

1、JVM 中判断两个 Class 对象是否为同一个类的条件：

1）类的<mark>全限定类名必须一致，包括包名</mark>；

2）加载<mark>类的 ClassLoader 必须相同</mark>。

也就是说，在 JVM 中即使两个 Class 对象来自同一个 `.class` 文件，被同一个虚拟机加载，若是 ClassLoader 对象不同，这两个对象就不相等。

<br>

2、JVM 必须知道一个类是由启动类还是用户自定义类加载器加载。

如果一个类是由用户自定义类加载器加载，那么 JVM 就会将这个类加载器的一个引用作为类信息的一部分保存到方法区中。当解析一个类到另一个类的引用时，JVM 需要保证者两个类的类加载器是相同的。

<br>

3、类的<mark>主动使用</mark>与<mark>被动使用</mark>

主动使用包括：

* 创建类实例；
  
* 访问某个类/接口的静态变量，或是对某个类的静态变量赋值；
  
* 调用类的静态方法；
  
* 反射，如 `Class.forName()`；
  
* 初始化一个类的子类；
  
* Java 虚拟机启动时被标明为启动类的类；
  
* Java 7 开始提供的动态语言支持。
  

被动使用：除了主动使用外的其他 Java 类的使用都可以看作是类的被动使用。



<br>

### 程序计数器

<mark>本质是一个指针</mark>，程序计数器可以看作是<mark>当前线程所执行的字节码的行号指示器</mark>，字节码解释器通过改变计数器的值来选取下一条需要执行的指令的字节码。

* 程序计数器本身仅占用一块非常小的空间，属于<mark>线程私有内存</mark>。

* 分支、循环、跳转、异常处理、线程恢复等功能都依赖程序计数器来完成。

* 如果执行的是一个 `native` 方法，则程序计数器为空。

> 程序计数器不会发生内存溢出（OOM）。

<br>

### 方法区

用于存储 JVM 加载的每个类的结构信息、运行时常量信息、静态变量、字符串字面量、数字常量、方法数据、构造函数、普通方法的字节码内容和即时编译器编译后的代码等数据。

- 方法区属于<mark>**线程共享**的内存区域</mark>；
- <mark>方法区也存在垃圾回收</mark>，但是很少，垃圾回收大多发生在堆内存。以下情况会进行垃圾回收：
  1. 常量池中的一些常量和符号引用若是没有被引用，则会被清理；
  
  2. 无用的类，所有实例被回收的类、ClassLoader 被回收的类、实例对象没有被引用的类。

> 在所有的 JVM 虚拟机中，方法区由 HotSpot 独有，J9 和 JRocket 没有。对于 HotSpot 虚拟机，可以将方法区理解为一个接口，永久带和元空间都是它的实现。

<br>

**方法区中的常量池**

1、类文件中常量池（*The Constant Pool*）

字节码文件记录当前这个类的所有相关信息，其中之一就是常量池。常量池存放编译器生成的各种字面量（Literal）和符号引用（*Symbolic References*）。

以下面这段字节码信息为例：

```java
Constant pool:
   #1 = Methodref          #6.#24         // java/lang/Object."<init>":()V 方法引用
   #2 = Fieldref           #25.#26        // java/lang/System.out:Ljava/io/PrintStream; 字段引用
   #3 = Class              #27            // jvm/ClassLoadTest 类信息引用
   #4 = String             #28            // hello, JVM 字符串引用
   #5 = Methodref          #29.#30        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #6 = Class              #31            // java/lang/Object
   #7 = Utf8               STR
   #8 = Utf8               Ljava/lang/String;
   #9 = Utf8               ConstantValue
  #10 = String             #32            // JVM
  // ...
  #28 = Utf8               hello, JVM // 字面量
```

其中第 `#1` 表示方法引用，`#28` 表示字符串字面量。

**字面量**，包括：

1）文本字符串；

2）八种基本类型的值；

3）被声明为 `final` 的常量等。

**符号引用**，包括：

1）类和接口的全限定名（*Fully Qualified Name*）；

2）字段的名称和描述符（Descriptor）；

3）方法的名称和描述符。

<br>

2、运行时常量池（*The Run-Time Constant Pool*）

运行时常量池是在类加载过程中，将 `.class` 文件中的常量池信息加载到内存中，并在 JVM 运行时使用的一个存储常量的表。

在加载类的过程中，类的常量池会被完全加载到内存中，并在内存中动态地生成一个运行时常量池。运行时常量池中存储了一些字面量和符号引用，用于支持类文件中的各种指令。

> **运行时常量池的位置**
> 
> 1）JDK 1.6 及之前版本，它位于**方法区的永久代中**；
> 
> 2）JDK 1.7 开始，将**字符串常量池移至堆区**。这里 JDK 文档并没有说运行时常量池是否也跟着移到堆区，也就是说**运行时常量依然在方法区**，永久代仍存在于 JDK 1.7 中；
> 
> 3）JDK 1.8，JVM 移除了永久区，取而代之的是元空间（Metaspace），也就是将本地内存用来存储。**字符串常量池还在堆**，运行时常量池还在方法区，只不过**方法区的实现从永久代变成了元空间。**
> 
<br>

3、字符串常量池

1）为了避免多次创建字符串对象，在 JVM 中开辟一块空间，储存**不重复**的字符串。

2）在直接使用双引号声明字符串的时候，会先去常量池找有没有相同的字符串，如果有则将常量池的引用返回给变量；如果没有，会在字符串常量池中创建一个对象，然后返回这个对象的引用。

3）使用 `new` 关键字创建，比如 `String a = new String("hello");` 这里**可能**创建两个对象。一个是用双引号括起来的 `hello` 字符串，另一个是 `new` 关键字在堆中创建的新对象。最终返回的是 `new` 创建的对象的地址。

<br>

> 在 JDK 1.7 之前，字符串常量池被放在**方法区** 中，实现是永久代（可以认为方法区是一个接口/规范，永久代和元空间都是它的实现）。因为永久代使用的是 JVM 的直接内存，空间有限，在大量使用字符串的场景下会导致 OOM 错误。
> 
> JDK 1.7 及以后，**字符常量池移至了堆区**，字符串的创建也在堆区。为了节省开支 `String#intern` 不再是把该字符串直接加入字符常量池，而是将其地址引用放到字符常量池。
> 
> 到了 JDK 1.8，JVM 使用元空间（MetaSpace）作为方法区的实现，来代替永久代。并将元空间从 JVM 内转移到本地内存上。JDK 1.8 的字符串常量池还是放在在堆区，运行时常量池在方法区。



<br>

### Java 栈

- 主管 Java 程序的运行。<mark>线程创建时创建</mark>，生命周期与线程相同，线程结束意味着栈内存释放；
- 属于<mark>线程私有的内存区域</mark>；
- <mark>栈不存在垃圾回收的问题，线程结束即释放内存</mark>；常见的报错有 `StackOverflowError/OutofMemory`，前者可能是因为数组下标越界，深层递归导致栈内存被耗尽；后者可能是创建的线程过多导致栈内存溢出。
- Java 8 中基本类型的变量、对象的引用变量和实例方法都是在函数的栈内存中分配的；
- 每个方法被执行时，都会创建一个<mark>栈帧</mark>，将栈帧压入栈中，当<mark>方法正常返回或是抛出未捕获的异常</mark>时，栈帧随之出栈。
  

<img src="./assets/image-20210404150846257.png" alt="image-20210404150846257" style="zoom:200%;" />

<br>

**栈中的内容**

- 栈帧用于<mark>存储局部变量表、操作数栈、动态链接、方法相关信息和方法返回值等信息</mark>。每个方法从调用直到执行完毕的过程，就对应着一个栈帧由入栈到出栈的过程；
  
- 栈中的数据都是以栈帧的格式存在，栈帧是一个内存区块，是一个数据集，是一个有关方法和运行期数据的数据集；
  
- 栈帧实际上就是一个个不同的 Java 方法；
  
- 栈帧中主要保存 3 类数据：
  
  1. 本地变量（*Local Variables*），输入参数和输出参数以及方法内的变量；
     
  2. 栈操作（*Operand Stack*），记录出栈入栈的操作；
     
  3. 栈帧数据（*Frame Data*），包括类文件、方法等。
     



<br>

### 本地方法栈

本地方法栈与 Java 栈的作用类似，区别是执行 Java 方法使用的是 Java 栈，而执行 `native` 方法时使用的是本地方法栈；本地方法栈属于<mark>线程私有的内存区域</mark>。



<br>

### 堆

用于存放对象实例，<mark>几乎所有</mark>的对象实例都在堆中分配内存。<mark>在 JVM 启动时创建</mark>，一个 JVM 实例只存在一个堆内存（堆内存的大小可调节），属于<mark>**线程共享**的内存区域</mark>。

加载器读取了类文件后，将类信息、方法、常量和变量放到堆内存中，保存所有引用类型的信息，以方便执行器执行。根据对象生命周期的不同，JVM 会把对象进行分代管理，由垃圾回收器进行分代垃圾回收。

> 堆中没有可用空间时，会抛出 OOM 异常。

<br>

#### 堆/栈/方法区交互

<img src="./assets/image-20210406171044900.png" alt="image-20210406171044900" style="zoom:100%;" />

<br>

#### 堆内存分配

堆内存分配有物理分配和逻辑分配之分。**物理上**只存在两个部分：

* 新生代（*Young/New Generation Space*），类创建/应用/销毁的区域，类在新生代被创建和应用，无引用后被垃圾收集器回收；新生代分为两部分：伊甸区（*Eden Space*）和幸存者区（*Survivor Space*）。
* 老年代（*Tenure/Old Generation Space*），占用内存大的对象或者在发生新生代中存活超过一定岁数的对象会进入老年代。

**逻辑上**分为三部分：

* 新生代；
* 老年代；
* Java 7 永久代（*Permanent/Perm Space*）/Java 8 元空间（*Meta Space*）

<br>

##### 新生代

1）伊甸区

- 几乎所有的类都是在伊甸区被创建出来；
- 新生代分配不了的大对象会直接进入老年代；

2）幸存者区，幸存者区分为：From 区（*Survivor From Space*）和 To 区（*Survivor To Space*）。From 和 To 不是固定的，每次 GC 后 From 和 To 会交换，GC 后为空的就是 To 区。新生代 GC（*Young/Minor GC*），会对新生区包括伊甸和幸存者区进行垃圾回收。

**新生代 GC 流程**

在第一次 GC 开始前，对象只会存在于 Eden 区和幸存者 From 区，幸存者 To 区是空的。当伊甸区空间用完且又需要创建对象时，垃圾回收器就会对伊甸区进行垃圾回收（*Minor GC*）。**只要产生  GC，伊甸区几乎会被全部清空，剩余的对象转移到幸存者 To 区**。

接着原先的 From 区变成 To 区，而 To 区变成新的 From 区。新对象依旧在伊甸区被创建，如果又发生 GC，对象继续在 From 和 To 中交换，当幸存者区中 From 和 To 来回复制交换次数达到 15 次之后，还存活的对象就存入养老区。次数由 JVM 参数 `MaxTenuringThreshold` 决定，默认为 15。

<br>

> 新生代的内存区域小于老年代，占堆内存的 1/3，对象存活率低，因此新生代 GC 适合使用复制算法。

<br>

![image-20210404164924346](./assets/image-20210404164924346.png)

<br>

##### 老年代

新生代分配不了的大对象或到达一定岁数后的对象都会进入老年代。老年代内存空间满了之后开启 *Full/Major GC*，<mark>FGC 会清理整个堆空间</mark>，包括新生代和老年代。FGC 多次后若老年代空间依然是满的则会报 OOM 异常：

```
java.lang.OutOfMemoryError:Java heap space
```

老年代产生 OOM 异常原因可能有：

* 堆内存设置不够，可通过参数 `-Xms` 和 `-Xmx` 来调整；

* 创建了大量对象，并且长时间不能被垃圾回收器收集（仍被引用）。

<br>

> 老年代内存区域相对较大，占堆空间的 2/3，对象存活率高，因此老年代 GC 算法常用标记清除或标记清除和标记整理的混合实现。

<br>

##### 堆参数调整

- `-Xmx`，设置初始分配大小，默认为物理内存的 1/64；

- `-Xms`，最大分配内存，默认为物理内存的 1/4；

- `-XX:MaxTenuringThreshold`，设置对象在新生代中交换的次数；

- `-XX:+PrintGCDetails`，输出详细的 GC 处理日志。



#### 永久代与元空间

##### 永久代

Java 7 的堆内存结构分为新生代、老年代和永久代。永久区是常驻内存的区域，用于存放 JDK 自身所携带的类和接口元数据。永久代存储的是运行时类信息，被装载进入此区域的数据是不会被垃圾回收的，关闭 JVM 才会释放此区域所占用的内存。

![image-20210404162929943](./assets/image-20210404162929943.png)



<br>

##### 元空间

Java 8 的堆内存结构分为新生代、老年代和元空间。其中元空间占用的是计算机本地内存，非 JVM 内存。

![image-20210404163110189](./assets/image-20210404163110189.png)

Java 8 的元空间并不在虚拟机堆内存中，而是在本机的物理内存。元空间的大小受本地内存限制，类的元数据放入本地物理内存中，此时可以加载多少类的元数据由本地内存的实际可用空间来控制。



<br>

#### 并发场景堆内存分配

在并发场景中，如何保证**内存分配过程**的线程安全性？如果两个线程先后把对象引用指向了同一个内存区域，怎么办？

先回答第一个问题，有两个办法：

1、对分配内存空间的动作做**同步处理**，采用 **CAS 机制**，配合失败重试的方式保证更新操作的线程安全性。缺点是每次分配时都需要进行同步控制，效率较低。

2、每个线程在 Java 堆中预先分配一小块“私有”内存，在给对象分配内存时，直接从这块内存中分配。当这部分区域用完之后，再重新分配一块“私有”内存给线程。

<br>

HotSpot 虚拟机中采用的是第 2 种方式，被称为 **TLAB 分配**（*Thread Local Allocation Buffer*），这部分 Buffer 是从堆中划分出来的，是**本地线程独享**的。虽然说 TLAB 是线程独享的，但只是在“分配”这个动作上是线程独占的。至于在读取、垃圾回收等动作上都是线程共享的，而且在使用上也没有什么区别。

TLAB 仅作用于新生代的伊甸区，是否使用 TLAB 是可以选择的，可以通过 `-XX:+/-UseTLAB` 参数来设置。

> 正是因为有 TLAB 的存在，所以之前在聊起[堆](#堆)的时候说：几乎所有的对象都是在堆中分配的，“几乎所有”而不是“所有”，因为存在一些对象会在 TLAB 中分配的情况。

<br>

使用 TLAB 之后，在 TLAB 上给对象分配内存时线程独享，并发场景下的冲突得到解决。**但是 TLAB 内存自身从堆中划分出来的过程也可能存在内存安全问题**，所以在对于 TLAB 的分配过程，还是需要进行同步控制的，但是这种开销相比于每次为单个对象划分内存时候对进行同步控制的要低得多。

<br>

TLAB 是线程特有的，它的内存区域不大，会出现不够用的情况，有 2 种解决办法：

1、直接在堆中分配，但存在极端情况：TLAB 只剩下 1 KB，导致后续可能大多数对象都需要直接在堆中分配；

2、废弃当前 TLAB，重新申请 TLAB 空间再次进行内存分配，会出现频繁的废弃 TLAB、频繁申请 TLAB 的情况。

<br>

为了解决这个问题，JVM 支持定义了一个 `refill_waste` 值，这个值可以看成**最大浪费空间**。当请求分配的内存大于 `refill_waste` 的时候，会选择在堆内存中分配。若小于 `refill_waste` 值，则会废弃当前 TLAB，重新创建 TLAB 进行对象内存分配。



<br>

## 垃圾回收

> 可以看看这篇文章，[Java Garbage Collection Basics](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html) 出自 Oracle。

### 强/软/弱/虚引用

1、**强引用**，即为平常使用得最多的引用，强引用的对象**即便在程序内存不足（OOM）时也不会被回收**；

2、**软引用**，对象在**程序内存不足时回收**；

3、**弱引用**，只要 JVM 的**垃圾回收器发现就会回收**；

4、**虚引用**，只要 JVM 的**垃圾回收器发现就会回收**。但在被**回收之前**会被放入回收队列 `ReferenceQueue`。其他几种引用是被 JVM 的垃圾回收机制**回收之后**，才放入 RefernceQueue 中。虚引用大多被用于引用销毁前的处理工作，因此虚引用创建的时候必须带有 RefernceQueue 参数。



<br>

### 垃圾收集触发

#### YGC

伊甸区内存满的时候触发，涉及整个新生代。

#### FGC

* 老年代空间不足。
  
* 创建占用连续内存空间大的对象。例如很长的数组，老年代可能有剩余空间，但是无法找到足够大的连续空间来分配给当前对象，会触发 FGC。
  
* 调用 `System.gc()` （只是对 JVM 发起 FGC 请求，至于何时触发，由 JVM 自行判断）。
  
* 方法区空间不足。当系统中要加载的类、反射的类和调用的方法较多时，方法区可能会被占满，在未配置采用 CMS 的情况下会执行 FGC。

<br>

**发生 FGC 需要怎么做？**

1、调整老年代空间大小；

2、对于一些不常用的大对象，使用完毕尽快销毁；

3、调整进入老年代的对象的 epoch；



<br>

### GC 日志信息

> **如何查看**
> 
> idea 的 vmoption 中添加参数
> 
> ```shell
-XX:+PrintGCTimeStamps -XX:+PrintGCDetails -Xloggc:./gc.log # Linux or macOS
> ```
> 
> ```shell
-XX:+PrintGCTimeStamps -XX:+PrintGCDetails -Xloggc:.\gc.log # Windows
> ```
> 

```java
public static void main(String[] args) {
    new Thread(() -> {
        byte[] arr1 = new byte[4 * 1024 * 1024];
        arr1 = new byte[8 * 1024 * 1024];
        arr1 = new byte[10 * 1024 * 1024];

        byte[] arr2 = new byte[4 * 1024 * 1024];
        arr2 = null;

        byte[] arr3 = new byte[4 * 1024 * 1024];
        byte[] arr4 = new byte[4 * 1024 * 1024];
        byte[] arr5 = new byte[4 * 1024 * 1024];
        byte[] arr6 = new byte[4 * 1024 * 1024];

        System.gc(); // 手动进行 FGC
    }).start();
    new Thread(() -> {
        while (1 == 1) {
            GCTest o = new GCTest();
        }
    }).start();
}
```

<img src="./assets/image-20210406150611786.png" alt="image-20210406150611786" style="zoom:200%;" />



<br>

### 垃圾回收算法

> **分代垃圾收集算法**，没有最好的算法，只有根据堆中不同代使用的不同算法。

<img src="./assets/image-20210406151057954.png" alt="image-20210406151057954" style="zoom:200%;" />

<br>

#### 对象回收判断

**引用计数法**

在 Java 中，引用和对象是有关联的。如果要操作对象则必须使用对象的引用进行操作，因此可以通过对象的引用次数来判断一个对象是否可以被回收。

可以给对象添加一个引用计数器，每当有一个地方引用它，计数器值加 1；每当有一个引用失效时，计数器值减 1。任意时刻计数器值为 0 的对象就是可回收对象。

缺点是每次对象赋值时都要维护引用计数器，计数器本身也有一定的消耗，且较难处理循环引用。

> JVM 不采用此方法，因为它很难解决对象之间相互循环引用的问题：假设对象 A 和 B 相互引用，此外没有其他引用指向 A 和 B。这种情况下，若 A 和 B 已不再被使用，但由于它们的引用计数器都不为 0，在使用引用计数法时，就会将这两个对象判断为不可被回收。
>
> <br>

**可达性分析**

*GC Roots*（垃圾收集器对象），垃圾回收过程中会回收不是 *GC Roots* 且没有被 *GC Roots* 引用的对象。通常有以下对象会被认为是 Root 对象：

1. 虚拟机栈中引用的对象（栈帧中的本地变量表）；

2. 方法区中静态属性引用的对象；

3. 方法区中常量引用的对象；

4. 本地方法栈中 `native` 方法引用的对象。

通过一系列 *GC Roots* 对象作为起点进行遍历，如果在 *GC Roots* 和某个对象之间没有可达路径，则表示该对象是不可达的。**不可达的对象不一定会成为可回收对象**：

1. 可能是进入 `DEAD` 状态的线程，而进入 `DEAD` 状态的线程是可以恢复的，GC 不会回收；

2. 不可达对象为 Root 对象，JVM 认为 Root 对象是不可回收的，并且 Root 引用的对象也是不可回收的。

<br>

#### 垃圾回收

**复制算法**（Copying），适用于新生代。新生代发生 YGC 时，使用的就是复制算法。因为新生代中的对象一般存活率不高，来回复制开销小。

复制算法需要使用到两块内存区域，每次只用其中一块，当一块用完，就将存活的对象复制到另一块。新生代对象一般存货率较低，所以通常使用两块 10% 的内存作为空闲和活动空间，即 From 和 To 区。另外 80% 的区域则用来给新建对象分配内存，即 Eden 区。

一旦发生 GC，JVM 会将 From 区中存活的对象与 Eden 区中存活的对象复制到 To 区，接下来将 Eden 和 From 这 90% 的内存全部释放。

此算法的优点是不会产生内存碎片；缺点是消耗内存，当有大量对象存活时，会产生较大复制开销。

![image-20210406162509076](./assets/image-20210406162509076.png)

<br>

**标记清除算法**（*Mark Sweep*），适用于老年代。老年代的 GC 算法一般是由标记清除或标记压缩实现。

标记清除算法分为标记和清除两个阶段，先**标记（Marking）出要回收的对象**，然后再统一回收/清除（Sweep）这些对象。

程序在运行期间，若可用内存被耗尽，GC 线程就会被触发。将程序暂停，将要回收的对象标记，最终统一回收被标记的对象，完成标记清除工作后应用程序恢复运行。

**优点**：不需要像复制算法一样消耗额外空间。

**缺点**：

1、标记和清除分两次扫描，耗时严重，效率较低；

2、会产生内存碎片，为了解决这个问题，JVM 需要维持一个内存的空闲列表，存在额外开销。而且在分配数组对象时，寻找连续的内存空间不好找。

<table>
  <caption>回收前</caption>
	<tr>
  	<td style="background-color:lightgreen;">存活</td>
    <td style="background-color:red;">可回收</td>
    <td>空闲</td>
  </tr>
  <tr>
  	<td>空闲</td>
    <td style="background-color:lightgreen;">存活</td>
    <td style="background-color:red;">可回收</td>
  </tr>
  <tr>
  	<td style="background-color:lightgreen;">存活</td>
    <td>空闲</td>
    <td style="background-color:lightgreen;">存活</td>
  </tr>
  <tr>
    <td>空闲</td>
    <td>空闲</td>
  	<td style="background-color:red;">可回收</td>
  </tr>
</table>

<br>

<table>
  <caption>回收后</caption>
	<tr>
  	<td style="background-color:lightgreen;">存活</td>
    <td>空闲</td>
    <td>空闲</td>
  </tr>
  <tr>
  	<td>空闲</td>
    <td style="background-color:lightgreen;">存活</td>
    <td>空闲</td>
  </tr>
  <tr>
  	<td style="background-color:lightgreen;">存活</td>
    <td>空闲</td>
    <td style="background-color:lightgreen;">存活</td>
  </tr>
  <tr>
    <td>空闲</td>
    <td>空闲</td>
  	<td>空闲</td>
  </tr>
</table>



<br>

**标记压缩**（*Mark Compact*），适用于老年代。**标记压缩 = 标记清除 + 标记整理**，经过多次标记清除 GC 后，才会进行压缩整理。

在标记整理的**压缩阶段**，不是标记回收对象，而是<mark>标记存活对象，把所有的存活对象都向一端移动，然后直接清除存活边界外的内存</mark>。标记存活的对象就得到了整理，按照内存地址依次排序，未被标记的内存就会被清理。在之后给新对象分配内存时，JVM 只需要持有空闲内存起始地址即可，比标记清除维护的空闲列表开销少。

**优点**：没有内存碎片。

**缺点**：需要付出移动对象的成本，相比于前几个算法耗时长。



![image-20210406165228525](./assets/image-20210406165228525.png)

<br>

#### GC 算法比较

> - 效率
>
>   复制 > 标记清除 > 标记压缩
>
> - 内存整齐度
>
>   复制 = 标记压缩 > 标记清除
>
> - 内存利用率
>
>   标记压缩 = 标记清除 > 复制
>

**算法适用场景**

- 新生代内存区域相对老年代小，占堆空间的 1/3，对象存活率低，因此 YGC 适合使用复制算法；

- 老年代内存区域相对较大，占堆空间的 2/3，对象存活率高，因此 FGC 算法通常用标记清除或标记压缩。



<br>

### **垃圾回收过程**

对象被判定是否可回收，需要经历两个阶段：

1、可达性分析。

2、查看需要回收的对象是否重写了 Object#finalize 方法，finalize 方法的作用是给该对象一次救活的机会。回收流程如下：

- JVM 进行垃圾回收前会查看需要回收对象是否重写了 finalize 方法，未重写直接进行垃圾回收；

- 若重写了则调用该对象的 finalize 方法（可能进行清理操作），不回收该对象；

- 若该对象的 finalize 方法已被调用过则对该对象进行垃圾回收操作。




<br>

### 编码过程的优化

* 不显式调用 `System.gc()`，减少 GC 的频率；

* 尽量减少临时对象的使用。在方法结束后，临时对象便成为了垃圾，减少临时变量就相当于减少垃圾产生；

* 对象不用时显式置为 `null`，为 `null` 的对象都会被作为垃圾处理；

* 尽量使用 `StringBuilder` 来代替 `String` 进行字符串累加。因为 `String` 的底层是 `final` 类型的数组，所以 `String` 的增加其实是建了一个新的 `String`；

* 允许的情况下尽量使用基本类型来替代包装类型，基本类型比包装对象占用的内存资源少；

* 合理使用静态变量。因为静态变量属于全局变量，不会被 GC 回收。



<br>

### 垃圾收集器

> 1. Serial；
>2. Parallel；
> 3. CMS；
>4. [G1](https://docs.oracle.com/en/java/javase/17/gctuning/garbage-first-g1-garbage-collector1.html)；
> 5. [ZGC](https://docs.oracle.com/en/java/javase/17/gctuning/z-garbage-collector.html)：是 JDK 11 中引入的一种全新的垃圾收集器，它是一种低延迟的垃圾收集器，可以在不超过 10ms 的时间内完成垃圾收集。与其他垃圾收集器不同的是，ZGC 收集器可以对整个堆内存进行垃圾收集，而不是只针对部分区域。
>
> 
>
> 更多内容可以查看 [Oracle HotSpot VM GC](https://docs.oracle.com/en/java/javase/17/gctuning/index.html)

<br>

#### Serial

Serial 收集器是一种单线程的垃圾收集器，只使用一个线程进行垃圾收集。在垃圾收集期间会暂停应用程序的所有线程（*Stop The World*），再开始进行垃圾收集，收集完毕恢复应用运行。它适用于小型或中小型的应用程序，或者是作为客户端模式下的默认垃圾收集器。



<br>

#### Parallel

Parallel 收集器是 Serial 收集器的多线程版本，它可以使用多个线程进行垃圾收集，提高垃圾收集的效率。适用于多核处理器和大型应用程序。

* ParNew 作用于新生代，是 Serial 收集器的多线程版本。

  > 主要适用于以前使用 CMS 垃圾收集器的应用程序，可以和 CMS 协同工作。

* *Parallel Scavenge* 作用于新生代，类似于 ParNew，不同的是 *Parallel Scavenge* 重点在提高吞吐量，主要适用于有大量可用 CPU 和较少内存的场景。

* *Parallel Old* 作用于老年代，是 *Parallel Scavenge* 的老年代版本。

<br>

**并发与并行**

![img](./assets/7000.jpeg)

<br>

#### CMS

CMS（*Concurrent Mark Sweep*） 收集器是一种并发垃圾收集器，以获取最短回收停顿时间为目标。它可以和应用程序同时运行，减少垃圾收集过程中的应用程序的停顿时间。



##### CMS 回收过程

1、初始标记（*CMS initial mark*)，标记需要回收的对象，会停止所有的用户线程（STW）。

2、并发标记（*CMS concurrent mark*），进行 *GC Roots Tracing*，追踪并记录可达对象（存活对象）。

3、重新标记（*CMS remark*），为了修正并发标记期间因用户程序继续运行而导致标记产生变动的那一部分对象的重新标记。这个阶段的停顿时间一般比初始标记阶段长。

4、并发清除（*CMS concurrent sweep*）。恢复用户线程，同时清理垃圾对象。

> 整个 GC 过程当中消耗时间最长的是并发标记和并发清除过程，但是这两个阶段的垃圾回收线程可以与用户线程并发执行。

<br>

##### CMS 缺点

- CMS 是基于标记清除算法，在清理的过程中会有大量的空间碎片。

- CMS 对 CPU 资源敏感，在并发阶段虽然不会停止用户线程，但是会占用一部分线程来进行垃圾回收。

- CMS 无法回收浮动垃圾。CMS 在并发清理阶段还可以运行用户线程，需要预留一部分空间提供并发收集时候的用户线程使用。这时候还会产生新的垃圾，而这部分垃圾 CMS 无法在本次回收掉，这部分就是浮动垃圾。

##### CMS 参数

| 参数                                 | 说明                                                         |
| ------------------------------------ | ------------------------------------------------------------ |
| `-XX:CMSInitiatingOccupancyFraction` | CMS 不像其他的收集器，老年代几乎全部满了再进行回收。CMS 在年老代使用了 68% 的空间后就会被激活，可以通过此参数指定激活空间大小。<br />如果 CMS 在运行时候预留的内存无法满足程序需要，就会出现一次 “Concurrent Mode Failure” 失败，这时候虚拟机临时启用 *Serial Old* 收集器重新来进行老年代的垃圾收集。 |
| `-XX:+UseCMSCompactAtFullCollection` | 在 FGC 完成后附加一个碎片整理过程，此碎片整理过程无法并发，会导致停顿时间变长。 |
| `-XX:+UseCMSCompactAtFullCollection` | 在执行多少次不压缩的 FGC 后，进行一次压缩的 FGC。            |




<br>

##### CMS 中 FGC 调整

> 1）创建占用连续内存空间大的对象，可以使用以下两个参数解决：
>
> * `-XX:+UseCMSCompactAtFullCollection` 
>
> * `-XX:CMSFullGCsBeforeCompaction`
>
> 2）老年代空间不足
>
> 如果执行 FGC 后空间仍然不足，则抛出错误：`java.lang.OutOfMemoryError: Java heap space`。因此应尽量做到让对象在 YGC 阶段被回收。
>
> 3）方法区空间不足
>
> 当系统中要加载的类、反射的类和调用的方法较多时，方法区可能会被占满，在未配置采用 CMS 的情况下会执行 FGC。如果经过 FGC 仍然无法回收，JVM 会抛出 OOM。可采用的方法为增大永久代/元空间内存或者使用 CMS GC。
>

<br>

#### G1

从 Java 7 开始使用，作为 CMS 的长期替代。G1 收集器是一个并行的、并发的、增量压缩的低暂停垃圾收集器，是一种面向服务端应用的垃圾收集器。它将堆内存划分为多个独立的区域，并且每个区域的大小相同，可以根据垃圾的分布情况，优先收集垃圾最多的区域。

<br>

##### G1 回收过程

1、初始标记。标记需要回收的对象，会停止所有的用户线程（STW）。

2、并发标记。进行 *GC Roots Tracing*，追踪并记录可达对象（存活对象）。

3、最终标记。标记需要回收的对象，STW 时间比初始标记长。

4、筛选回收。对每个可回收的内存区域进行价值评估，会先回收价值较大的内存区域。

<br>

##### G1 优缺点

1）支持并行与并发，缩短 STW 停顿时间。

2）分代收集：保留了分代的概念。采用不同的方式去处理新创建的对象和经过多次 GC 的旧对象，以获取更好的收集效果。

3）空间整合：与 CMS 的标记清理算法不同，G1 整体上（FGC）看是基于标记整理算法实现的；从局部上（YGC）来看是基于复制算法实现的。

> G1 在整体上：将堆内存划分为多个大小相等的区域，在收集垃圾时，会从价值最高的区域开始回收，这个过程被称为“混合垃圾收集”（Mixed GC），也就是标记整理算法的应用。
>
> 在局部上来看：基于复制算法的策略，它将每个分区分为多个大小相等的小块，每个小块都会被标记为可用或不可用。在 YGC 中，G1 收集器使用了复制算法，将 Eden 区域中的存活对象复制到 To 区。在 FGC 中，由于每个分区都可能包含一部分存活对象和垃圾对象，所以使用标记整理算法。这种混合使用复制算法和标记整理算法的方式既能保证效率，又能保证回收效果。
>
> 因此，可以说 G1 垃圾收集器是基于标记整理算法实现的，但也采用了复制算法。

4）可预测的停顿：这是 G1 相对于 CMS 的另一个大优势，降低停顿时间是 G1 和 CMS 共同的关注点，但 G1 除了追求低停顿外，还能建立可预测的停顿时间模型。

<br>

> 关于 G1 还可以看看[这篇文章](https://tech.meituan.com/2016/09/23/g1.html)。



<br>

## 参考

[jdk 1.7 后 intern 方法的变化](https://blog.csdn.net/zzzgd_666/article/details/87999870)

[JVM 之 G1 和 CMS](https://blog.csdn.net/zsj777/article/details/80353464)

[CMS 发生 FullGc 分析](https://blog.csdn.net/peter7_zhang/article/details/107011297)

[JVM 之垃圾回收机制（GC）](https://juejin.cn/post/7123853933801373733)

https://mp.weixin.qq.com/s/9rYhlsGiWwvhg6aD8NIWEQ

