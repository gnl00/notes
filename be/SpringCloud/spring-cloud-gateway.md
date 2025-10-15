# spring-cloud-gateway

## 反向代理

在 Spring Cloud Gateway 中，反向代理功能是其核心能力之一，其底层基于 **WebFlux 的响应式编程模型**。整个请求的处理流程可以分为几个关键阶段，涉及多个核心类和组件。

---

### 一、与反向代理相关的核心类

1. **`HttpWebHandlerAdapter`**
    - 这是 Spring WebFlux 中的入口类，负责将原始 HTTP 请求封装为 `ServerWebExchange` 对象，作为整个 Gateway 处理流程的上下文。

2. **`RoutePredicateHandlerMapping`**
    - 负责根据配置的路由规则（`RouteDefinition`）匹配当前请求，找到对应的 `Route`。

3. **`Route`**
    - 表示一个路由规则，包含 ID、断言（Predicates）、过滤器（Filters）和目标 URI（即反向代理的目标地址）。

4. **`FilteringWebHandler`**
    - 获取匹配路由中的所有 GatewayFilter，并与全局过滤器（GlobalFilter）合并、排序，组成一个过滤器链（`GatewayFilterChain`）。

5. **`NettyRoutingFilter`**（关键反向代理执行类）
    - 是 Gateway 内置的 **核心过滤器（Core Filter）** 之一，负责实际执行 HTTP 请求的转发（即反向代理）。它使用底层的 **Reactor Netty HttpClient** 向目标服务发起请求，并将响应写回客户端。

6. **`GatewayFilterChain`**
    - 过滤器链，控制请求在 pre 阶段（转发前）和 post 阶段（转发后）的处理逻辑。

---

### 二、请求到达后反向代理的执行逻辑

一个请求进入 Spring Cloud Gateway 后，其执行流程大致如下：

1. **接收请求**  
   客户端请求首先由 WebFlux 的 `HttpWebHandlerAdapter` 接收，并封装为 `ServerWebExchange` 对象。

2. **路由匹配**  
   `RoutePredicateHandlerMapping` 遍历所有已加载的路由（`Route`），根据路由断言（如 Path、Method、Header 等）匹配当前请求，找到最合适的路由。

3. **构建过滤器链**  
   `FilteringWebHandler` 将匹配到的路由中的 GatewayFilter 与全局 GlobalFilter 合并，并按 `Ordered` 接口排序，形成一个有序的 `GatewayFilterChain`。

4. **执行 Pre 阶段过滤器**  
   在真正转发请求前，依次执行所有过滤器的 “pre” 逻辑（例如鉴权、日志记录、修改请求头等）。

5. **执行反向代理（核心转发）**  
   当执行到 `NettyRoutingFilter` 时，它会：
    - 根据 `Route` 中的 URI 构造目标请求；
    - 使用 Reactor Netty 的 `HttpClient` 向目标服务发起异步非阻塞的 HTTP 请求；
    - 获取目标服务的响应。

6. **执行 Post 阶段过滤器**  
   在收到目标服务响应后，依次执行所有过滤器的 “post” 逻辑（例如修改响应头、记录响应日志、异常处理等）。

7. **写回响应**  
   最终将处理后的响应通过 `ServerWebExchange` 写回给客户端。

---

整个流程体现了典型的 **责任链模式 + 响应式编程** 的设计思想，其中 `NettyRoutingFilter` 是实现反向代理功能的关键类 。

## VS Nginx

**Spring Cloud Gateway 的工作方式确实不同于 Nginx 这类传统反向代理**，但它 **仍然属于反向代理（reverse proxy）的范畴**，更准确地说，它是一种 **“应用层反向代理”** 或 **“API 网关型反向代理”**。

---

### 一、传统反向代理（如 Nginx） vs Spring Cloud Gateway

| 特性 | Nginx（传统反向代理） | Spring Cloud Gateway |
|------|----------------------|------------------------|
| **代理模型** | **透传式（pass-through）**：修改请求头/URL 后直接转发 TCP/HTTP 流，不解析完整请求体（除非配置） | **主动式（active proxy）**：完整接收请求 → 处理逻辑（过滤器）→ 用 HttpClient 主动发起新请求 → 接收响应 → 写回客户端 |
| **是否解析请求体** | 默认不解析，流式转发 | 会完整解析（除非配置为 streaming） |
| **编程模型** | 声明式配置（nginx.conf） | 响应式编程（Java + WebFlux） |
| **扩展能力** | 通过 Lua 或模块扩展 | 通过 Java 代码自定义过滤器、路由、断言等 |

---

### 二、为什么 Spring Cloud Gateway 仍被称为“反向代理”？

尽管实现机制不同，但 **其核心职责与反向代理一致**：

> “Spring Cloud Gateway is a reverse proxy. A reverse proxy is an intermediate server that sits between the client trying to reach a resource” .

关键在于 **“客户端无感知地访问后端服务”** 这一本质特征。无论是 Nginx 直接转发流量，还是 Gateway 主动发起新请求，**对客户端而言，它只和 Gateway 通信，不知道后端真实服务的存在**——这正是反向代理的定义。

此外，官方和社区普遍将其归类为反向代理：
- “Cloud Gateway is a Docker-packaged reverse proxy built on Spring Cloud Gateway”
- “I'm using Spring Cloud Gateway as a reverse proxy”
- “The API gateway acts as a reverse proxy accepting all incoming API calls”

---

### 三、更准确的定位：API 网关是反向代理的超集

实际上，**API 网关（如 Spring Cloud Gateway）是反向代理的一种高级形式**：

> “An API Gateway is a more specialized type of reverse proxy designed to manage, route, and orchestrate API calls” .

> “API Gateway is a reverse proxy that can be configured dynamically via API” .

也就是说：
- 所有 API 网关都是反向代理；
- 但并非所有反向代理都是 API 网关。

Spring Cloud Gateway 在反向代理的基础上，增加了：
- 动态路由
- 熔断、限流
- 认证鉴权
- 请求/响应转换
- 日志监控等能力

---

### 结论

**Spring Cloud Gateway 是反向代理**，只是其实现方式是 **“应用层主动代理”** 而非 **“网络层透传代理”**。  
它比 Nginx 更重、更灵活，适合微服务架构下的 **API 网关场景**；而 Nginx 更适合高性能、低开销的 **通用反向代理或负载均衡** 场景。

两者不是替代关系，而是常 **配合使用**：Nginx 作为 L7 入口负载均衡 + SSL 终止，Gateway 作为业务层 API 网关。