# Java IO

> It’s been a long time no dive into Java source code…



在使用 Java IO 操作之前，需要明确：

* **读还是写**

  源：InputStream、Reader

  目的：OutputStream、Writer

* **字节还是字符**（文本）

  源：InputStream（字节）、Reader（字符）

  目的：OutputStream（字节）、Writer（字符）

* **操作位置**

  源：

  * 硬盘：文件（File）；
  * 内存：数组、字符串；
  * 硬盘：`System.in`
  * 网络：`Socket`

  目的

  * 硬盘：文件（File）；
  * 内存：数组、字符串；
  * 硬盘：`System.out`
  * 网络：`Socket`

* **额外功能**

  * 流转换：InputStreamReader、OutputStreamWriter
  * 缓冲：BufferedXXX
  * 多个源：SequenceInputStream 序列流
  * 对象序列化：ObjectInputStream、ObjectOutputSrteam
  * 操作基本数据：DataInputStream、DataOutputStream

…

| 分类       | 字节输入流           | 字节输出流            | 字符输入流        | 字符输出流         |
| ---------- | -------------------- | --------------------- | ----------------- | ------------------ |
| 抽象类     | InputStream          | OutputStream          | Reader            | Writer             |
| 访问字符串 |                      |                       | StringReader      | StringWriter       |
| 访问文件   | FileInputStream      | FileOutputStream      | FileReader        | FileWriter         |
| 访问数组   | ByteArrayInputStream | ByteArrayOutputStream | CharArrayReader   | CharArrayWriter    |
| 访问管道   | PipedInputStream     | PipedOutputStream     | PipedReader       | PipedWriter        |
| 缓冲流     | BufferedInputStream  | BufferedOutputStream  | BufferedReader    | BufferedWriter     |
| 转换流     |                      |                       | InputStreamReader | OutputStreamWriter |
| 对象流     | ObjectInputStream    | ObjectOutputStream    |                   |                    |
| 打印流     |                      | PrintStream           |                   | PrintWriter        |
| 特殊流     | DataInputStream      | DataOutputStream      |                   |                    |



## 包结构

```
java/io
├── BufferedInputStream.java      ├── BufferedWriter.java
├── BufferedOutputStream.java     ├── ByteArrayOutputStream.java
├── ByteArrayInputStream.java     ├── CharArrayWriter.java
├── Closeable.java                ├── DataInputStream.java
├── DataInput.java                ├── DataOutputStream.java
├── DefaultFileSystem.java        ├── FileDescriptor.java
├── File.java                     ├── FileFilter.java
├── FileInputStream.java          ├── FileNotFoundException.java
├── FileOutputStream.java         ├── FileReader.java
├── FileSystem.java               ├── FileWriter.java
├── FilenameFilter.java           ├── FilterInputStream.java
├── FilterOutputStream.java       ├── FilterWriter.java
├── Flushable.java                ├── IOError.java
├── IOException.java              ├── InputStreamReader.java
├── ObjectInput.java              ├── ObjectInputStream.java
├── ObjectOutput.java             ├── ObjectOutputStream.java
├── ObjectStreamClass.java        ├── ObjectStreamField.java
├── OptionalDataException.java    ├── OutputStream.java
├── OutputStreamWriter.java       ├── PipedOutputStream.java
├── PipedInputStream.java         ├── PipedReader.java
├── PrintStream.java              ├── PrintWriter.java
├── RandomAccessFile.java         ├── SequenceInputStream.java
├── StringReader.java             ├── Writer.java
...
```



## 基础接口和抽象类

从接口和抽象说起，`java.io` 包中包含以下接口：

* Closeable

* Serializable 表明类可序列化/反序列化

  如果一个父类不可序列化，不可序列化类的子类可以正常进行序列化和反序列化。在序列化过程中，不会为不可序列化超类的字段写入数据。在反序列化过程中，不可序列化超类的字段将使用第一个（最底层）不可序列化超类的无参数构造函数进行初始化。

* DataInput 负责输入（读）数据

  `reading bytes from a binary stream and reconstructing from them data in any of the Java primitive types.` 从**二进制流**中读取数据，并转换成 Java 原始类型数组

* DataOutput 负责输出（写）**二进制流**数据

  `converting data from any of the Java primitive types to a series of bytes and writing these bytes to a binary stream.` 将 Java 原始类型转换成字节并以二进制流的形式输出

* Flushable

  可以这么理解，`is a data that can be flushed to the underlying stream` ` 执行 Flushable#flush 方法**真正**将数据流输出

* ObjectInput 继承自 DataInput，DataInput 主要负责处理 Java 原始数据类型，ObjectInput 则进行了补充，可以接收（处理） Object/Array/String 输入

* ObjectOutput 继承自 DataOutput 处理 Object/Array/String 输出

* InputStream 表示所有**字节输入流**类的超类，子类需要在 InputStream#read 方法中返回 `next byte of input` （即下一个输入的字节长度）

* OutputStream **字节输出流**，输出字节并将其发送到目的地。子类必须实现至少一个输出一个字节的输出方法

* ObjectInputFilter 进行数据输入时，过滤可进行反序列化的类

* Writer 写入**字符流**的抽象类

* Reader 读取**字符流**的抽象类

* FileSystem 本地文件系统抽象的抽象类

* FileFilter 过滤出绝对路径下的文件

* FilenameFilter 过滤出绝对路径下的文件名

* FilePermission 顾名思义，`holds filepath and file permissions`

* …



> 大概分为两部分：二进制字节流和字符流。

## 字节流基础类

字节流包装

* FilterInputStream/FilterOutputStream 是 InputStream/OutStream 的简单包装

字节流缓冲

* BufferedInputStream 为输入字节流增加缓冲区，并支持 mark/reset 方法。当 BufferedInputStream 进行实例化时，其内部也会同时创建一个 buffer 数组，在读取或跳过数据流中的字节时，内部缓冲区会根据需要从包含的输入数据流中重新填充，每次填充多个字节。mark 操作能记住当前的 position，reset 能将 position 回复到上一次 mark 的位置，可以实现重复读取
* BufferedOutputStream 该类实现缓冲输出流，以流的形式输出，不必为写入的每个字节调用底层系统方法。
* …



## 字符流基础类

字符流包装

* FilterWriter/FilterReader

字符流缓冲

* BufferedReader/BufferedWriter



## 字节字符组合类

* InputStreamReader
* OutputStreamWriter



## 流缓冲

带有 Buffer 关键字的类都是缓冲相关的类，为流操作提供了 mark（标记当前读位置） 和 reset（回到最近的读位置） 操作。

在使用 Buffer 时需注意，每一个 Buffer 都有一个表示缓冲池大小的变量

```java
private static final int DEFAULT_BUFFER_SIZE = 8192;
```

以 BufferedOutputStream 为例，只有缓冲池中的内容大小满足 `DEFAULT_BUFFER_SIZE`，他才会真正的输出，否则会暂时先保存在缓冲中，等到满足 `DEFAULT_BUFFER_SIZE` 才输出。当然也可以调用 `flush` 方法强制输出。



## 字节数组流

ByteArrayOutputStream/ByteArrayInputStream，使用方法大概如下：

```java
public class BASMain {
    public static void main(String[] args) throws IOException {
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        // 还可以使用 DataOutputStream 和 DataInputStream 配合
        // DataOutputStream dataOutputStream = new DataOutputStream(baos);
        baos.write(1);
        baos.write(2);
        baos.write("hello".getBytes(StandardCharsets.UTF_8));
        // dataOutputStream.writeInt(2);
        // dataOutputStream.writeUTF("world");
        byte[] buff = baos.toByteArray();
        /* 查看 buff 内容
        for (int i = 0; i < buff.length; i++) {
            System.out.println((char) buff[i]);
            System.out.println(buff[i] + " ");
        }
        */

        ByteArrayInputStream bais = new ByteArrayInputStream(buff);
        int byteResult = 0;
        while ((byteResult = bais.read()) != -1) {
            System.out.println((char) byteResult); // 转成 char 字符输出，可以看到“  hello”
            System.out.println(byteResult + " "); // 转成十进制输出，可以看到 1 2
            // bais.read(output, len, output.length - len);
        }
        // DataInputStream dataInputStream = new DataInputStream(bais);
        // System.out.println(dataInputStream.readInt());
        // System.out.println(dataInputStream.readUTF());
    }
}
```





## 参考文档

* https://www.cnblogs.com/yichunguo/p/11775270.html
