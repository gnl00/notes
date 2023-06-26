# Java 进阶

## SPI

**是什么**

SPI 全称 Service Provider Interface 服务提供接口。以数据库访问接口为例，Java 定义了访问接口 `java.sql.Driver` ，并规定了常用的方法。因为存在如 MySQL/PostGreSQL 等不同的数据库，其中数据库连接操作的实现不同的，Java 管不过来那么多实现。

所以 Java 说，我把 `java.sql.Driver` 接口暴露来，让你们外部各种各样的数据库们（也就是服务提供者）自己来实现，实现类必须带一个无参构造方法；之后在 `classpath:META-INF/services` 目录下，名字为 `java.sql.Driver` 的文件，将实现类的全限定类名保存在里面。

当服务调用者需要访问数据库的时候，不关心连接具体是如何实现的。只需要导入对应的 jar 包，Java 程序内部会利用 `java.util.ServiceLoader` 工具类从对应的位置加载到对应的数据库实现类。

如果想连接 MySQL 数据库，通过 `DriverManager` 加载 MySQL 的驱动，即可调用 `java.sql.Driver` 接口的对应方法。其中，`DriverManager` 内部使用的就是 `java.util.ServiceLoader` 来加载 MySQL 驱动实现类。之后如果想换成别的数据库，只需要将 jar 包替换即可，内部代码无需做大改动。

在上面的例子中存在着 3 个角色，服务接口定义（Java），服务提供者（各种数据库），服务调用者（客户端），SPI 的实现和使用就是围绕这三者展开的。



**SPI 和 API 的区别**

* 和 SPI 不同，SPI 是接口的定义和实现分开进行的，API 则是接口的定义和实现捆绑在一起的；

* 使用 SPI 的需要客户端加载对应的实现，而调用 API 服务则不需要。



**How to use**

```java
public interface CusDriver {
    void load();
}

public class CusDriverMySQL implements CusDriver {
    @Override
    public void load() {
        System.out.println("MySQL driver loaded");
    }
}

public class CusDriverPgSQL implements CusDriver{
    @Override
    public void load() {
        System.out.println("PgSQL driver loaded");
    }
}

// META-INF/spi.CusDriver
spi.CusDriverMySQL
spi.CusDriverPgSQL

public class SPITest {
    public static void main(String[] args) {
        ServiceLoader<CusDriver> serviceLoader = ServiceLoader.load(CusDriver.class);
        Iterator<CusDriver> iterator = serviceLoader.iterator();
        while (iterator.hasNext()) {
            System.out.println(iterator.next());
        }
    }
}
```



<br>

## JNDI

**是什么**

JNDI（Java Naming and Directory Interface），即 Java 命名服务和目录接口。是 Java 的一个标准 API，用于管理和访问各种命名和目录服务。

> 首先明确一点：JNDI 是一套 Java 提供的 API。

它允许 Java 应用程序使用统一的方式来访问不同的命名和目录服务，无论这些服务是基于文件、LDAP（轻量级目录访问协议）、DNS（域名系统）还是其他协议实现的。

举个例子，假设有一个 Web 应用程序，需要使用一个数据库。通常情况下，需要在应用程序代码中指定数据库连接信息，如主机名、端口号、用户名和密码等。但是这种做法有一些缺点，比如数据库连接信息可能会因为环境的变化而需要修改，如果连接信息写死在代码中，则每次修改连接信息都需要重新编译和部署应用程序。

JNDI 提供了一种更加灵活和可配置的方式来解决这个问题。你可以将数据库连接信息存储在 JNDI 命名空间中，应用程序只需要从 JNDI 中获取连接信息即可，无需知道连接信息的具体细节。

总的来说，JNDI 是 Java 平台的一个标准 API，提供了一种通用的、统一的方式来访问各种命名和目录服务。它可以将应用程序与特定的命名和目录服务的实现分离开来，提高应用程序的灵活性和可配置性。



<br>

假设有一个 Java 应用程序，需要访问一个名为 "jdbc/mydb" 的数据库连接池。可以先在 Web 服务器上配置这个连接池，然后在应用程序中使用 JNDI 查找这个连接池，从而获得连接池的引用。

```java
// 获取 JNDI 上下文
Context initCtx = new InitialContext();

// 查找数据库连接池
DataSource dataSource = (DataSource) initCtx.lookup("java:comp/env/jdbc/mydb");

// 获取数据库连接
Connection connection = dataSource.getConnection();
```

在上面的示例中，首先获取了 JNDI 上下文，然后使用 `lookup` 方法查找名为 "java:comp/env/jdbc/mydb" 的对象，也就是预先在 Web 服务器上配置好的数据库连接池。最后，使用连接池获取了一个数据库连接。

在查找 JNDI 对象时，需要指定一个 JNDI 名称，这个名称通常是由 "java:" 前缀和一个特定的名称空间组成。在上面的示例中，"comp/env" 是 Web 应用程序的默认名称空间，"jdbc/mydb" 是配置的 JNDI 名称。

<br>

## 参考

[Java SPI思想梳理](https://zhuanlan.zhihu.com/p/28909673)

