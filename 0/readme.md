# 0

<p>

- 2025.11

  [现代 CPU 性能分析与优化](https://weedge.github.io/perf-book-cn/zh/)

<p>

  [centrifugo-java](https://github.com/gnl00/one-springcloud)
  - 基本使用
  - 设计原理
  - 架构

<p>

- 2025.10

  [springcloud-gateway](https://github.com/gnl00/one-springcloud)
  - 基本使用
  - 反向代理实现逻辑

  <br>mybatis源码
  - PooledDataSource/PooledDataSourceFactory/PoolState/PooledConnection
  - cache模块实现（顺便实现了一遍LRUCache）

  <br>[dubbo源码](https://github.com/gnl00/dubbo-source)

<p>

- 2025.09

  [one-java-raft](https://github.com/gnl00/one-java-raft)。
  - 抽象出基于 netty 的网络层和 raft 逻辑实现的节点层
  - 实现 raft 算法的 leader 选举部分
  - 思考：oraft 中基于 netty 的客户端是否需要池化？（在实现 mini-mq 时客户端的连接是池化的）

  <br>[jishang-app](https://github.com/gnl00/app-jishang)
  - claude-code 和 codex 编程能力的一次测试
  - 虽然说生成的代码在实现上可能比较复杂，但是基本的功能实现和比较广为人知的业务逻辑实现还是完成得不错的
  - 总的来说 UI 界面的实现在当前 AI 代码生成来说还是存在瓶颈，只能看后续的升级或者借助类似 figma-mcp 这样的工具来作为辅助

<p>

- 2025.08

  [mini-mq](https://github.com/gnl00/mini-mq)
  - 实现一个简单的 mq
  - mq-client 的池化思维
  - MQClientServiceImpl#executeWithClient 方法中将 Client 作为参数传递操作做法

  <br>[mini-nacos](https://github.com/gnl00/mini-nacos)
  - 实现一个简单的配置中心
  - DeferredResult 异步返回的使用
  - Spring:**PropertySource/IOC相关类的配合使用**

<p>

- 2025.07

  [at-i](https://github.com/gnl00/-i-ati)
  - an ai api client based on electron

<p>

- 2025.06

  [one-mini-spring](https://github.com/gnl00/one-mini-spring)
  - 实现一个简单的 spring
  - 实现：IOC容器/Bean创建/**AOP**

  <br>[mini-puppy](https://github.com/gnl00/mini-puppy)
  - 实现一个简单的 tomcat
  - DeferredResult 异步返回的使用
  - Spring:**PropertySource/IOC相关类的配合使用**

<p>

- 2025.04

  redis延迟双删实现[springboot-data-redis](https://github.com/gnl00/springboot-data-redis)