# Centrifugo

> Centrifugo 是一个面向实时消息分发的服务端组件，常用于 WebSocket 场景下的广播、房间消息、在线状态、事件推送。

## 为什么使用 Centrifugo

和 Kafka / RocketMQ 等 MQ 相比，Centrifugo 更聚焦“客户端实时连接与分发”这件事，适合下面这类需求：

- Mobile/Web/Desktop 直接连到消息服务；
- 支持一对一、一对多消息；
- 支持 WebSocket；
- 服务端通过 HTTP API 推送消息。

## 架构总览

```text
业务服务 --(HTTP/GRPC publish)--> Centrifugo --(WebSocket/SSE)--> Client
                                     |
                                     +--> Redis/NATS/Kafka Engine（可选）
```

Centrifugo 通常作为“实时网关”存在：

- 业务服务负责鉴权、业务计算、事件生产；
- Centrifugo 负责连接管理、订阅管理、消息路由与分发；
- 客户端负责订阅频道并处理实时事件。

架构优势：

- 鉴权、订阅权限、风控审核都能走统一的后端服务
- App/Web/PC 可统一用无需插件本身就支持的 WebSocket/SSE/REST 消息接收发送方式
- 扩展方便，Centrifugo 本身“无状态” 方便多实例部署 + 存储 Broker 可扩展

## 快速启动

### 1. 生成配置

```shell
docker run --rm -v$PWD:/centrifugo centrifugo/centrifugo:v6 centrifugo genconfig
```

### 2. 开启管理后台（可选）

编辑 `config.json`：

```json
{
  "admin": {
    "enabled": true
  }
}
```

### 3. 启动服务

```shell
docker run --rm --ulimit nofile=262144:262144 \
  -v config/file/path:/centrifugo \
  -p 8000:8000 centrifugo/centrifugo \
  centrifugo -c config.json
```

访问 `http://localhost:8000` 可进入管理页。

## 连接与订阅

### 1. 生成连接 Token

```shell
# 为用户 123722 生成 token
centrifugo gentoken -u 123722
```

### 2. 前端连接与订阅示例

```html
<script src="https://unpkg.com/centrifuge@5.4.0/dist/centrifuge.js"></script>
<script>
  const centrifuge = new Centrifuge("ws://localhost:8000/connection/websocket", {
    token: "<PUT-YOUR-TOKEN-HERE>"
  });

  centrifuge
    .on("connecting", (ctx) => console.log(`connecting: ${ctx.code}, ${ctx.reason}`))
    .on("connected", (ctx) => console.log(`connected over ${ctx.transport}`))
    .on("disconnected", (ctx) => console.log(`disconnected: ${ctx.code}, ${ctx.reason}`))
    .connect();

  const sub = centrifuge.newSubscription("channel");
  sub
    .on("publication", (ctx) => console.log("message", ctx.data))
    .on("subscribed", () => console.log("subscribed"))
    .on("unsubscribed", (ctx) => console.log(`unsubscribed: ${ctx.code}, ${ctx.reason}`))
    .subscribe();
</script>
```

如果出现 `permission denied`，需要在服务端允许客户端直接订阅：

```json
{
  "channel": {
    "without_namespace": {
      "allow_subscribe_for_client": true
    }
  }
}
```

## 消息历史（history）

开启频道历史能力：

```json
{
  "channel": {
    "without_namespace": {
      "history_size": 10,
      "history_ttl": "60s"
    }
  }
}
```

允许客户端/订阅者拉取历史：

```json
{
  "channel": {
    "without_namespace": {
      "allow_history_for_subscriber": true,
      "allow_history_for_client": true
    }
  }
}
```

## 频道设计建议

### 房间频道（room channel）

适用：同一消息要广播给一组用户。

优点：

- 服务端一次发布，成本低；
- 吞吐更高，延迟更稳定；
- 实现简单。

缺点：

- 个性化内容处理较弱；
- 权限变更与踢人逻辑要更严格。

### 用户私有频道（user channel）

适用：每个用户消息内容或权限不同。

优点：

- 支持个性化 payload；
- 审计和权限粒度更细。

缺点：

- 发布放大（N 用户 -> N 次发送）；
- 服务端复杂度和资源成本更高。

### 实战策略

- 广播为主：优先 room channel；
- 个性化为主：优先 user channel；
- 混合场景：room 发通用消息，user channel 发个性化补丁。

### 官方推荐场景（大量群订阅）

在“一个用户可能加入很多群”的场景下，官方更推荐方案 2（用户私有 channel）：

- 客户端尽量只订阅少量频道（通常是个人频道），减少订阅数量和重连恢复成本；
- 群消息由后端按成员关系 fanout 到多个用户个人频道；
- 用户退群时，只需停止向该用户 personal channel 分发该群消息。

这种方案的核心收益是连接与订阅资源更容易管理，代价是服务端 fanout 开销会上升。超大群场景通常需要在业务后端与 Centrifugo 之间增加队列缓冲。

## FAQ

- How many connections can one Centrifugo instance handle?

  This depends on many factors. Real-time transport choice, hardware, message rate, size of messages, Centrifugo features enabled, client distribution over channels, compression on/off, etc.

  Generally, we suggest not putting more than 50-100k clients on one node - but you should measure for your use case.

  [Million connections with Centrifugo](https://centrifugal.dev/blog/2020/02/10/million-connections-with-centrifugo)

- Memory usage per connection?

  Depending on transport used and features enabled the amount of RAM required per each connection can vary.

  For example, you can expect that each WebSocket connection will cost about 30-50 KB of RAM, thus a server with 1 GB of RAM can handle about 20-30k connections.

- How can I know a message is delivered to a client?

  You can, but Centrifugo does not have such an API. What you have to do to ensure your client has received a message is sending confirmation ack from your client to your application backend as soon as the client processed the message coming from a Centrifugo channel.

  It means Centrifugo has no embedded ACK support, you need to implement it yourself in your backend service. Centrifugo 未内置 ACK 支持，需要在自己的服务中实现。

## 参考

- https://github.com/gnl00/centrifugo-java
- https://centrifugal.dev/docs/getting-started/quickstart
- https://centrifugal.dev/docs/server/history_and_recovery
- https://centrifugal.dev/docs/faq
- https://centrifugal.dev/docs/tutorial/centrifugo
