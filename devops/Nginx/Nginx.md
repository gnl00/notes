# Nginx

> Nginx（Ngine X），是一个高性能，高可用的Web和方向代理服务器，可用于 HTTP、HTTPS、SMTP、POP3 和 IMAP 协议。它的设计不像传统的服务器一样使用线程处理请求，而是采用事件驱动机制，即一种异步的时间驱动结构。

<br>

## 特点

- 高并发，高性能。Nginx 可以比其他服务器更快地响应请求
- 模块化架构，且模块之间耦合度低，使得它的扩展性非常好
- 异步非阻塞的事件驱动模型，与 `Node.js` 类似
- 支持热部署、平滑升级，它可以连续几个月甚至更长的时间不需要重启服务器，具有高可靠性
- 低内存消耗

<br>

## 操作

```shell
# 启动服务
start nginx 

# 修改配置后重新加载生效
nginx -s reload

# 重新打开日志文件
nginx -s reopen

# 测试配置文件是否正确
nginx -t -c /path/to/nginx.conf 

# 验证配置是否正确
nginx -t

# 查看版本号
nginx -V

# 快速停止或关闭
nginx -s stop

# 正常停止或关闭
nginx -s quit
```



<br>

## 功能

### 处理静态资源

> 处理静态资源服务（动静分离），动静分离是指在 Web 服务器架构中，将静态页面与动态页面或者静态内容接口和动态内容接口分开不同系统访问的架构设计方法，进而提升整个服务的访问性和可维护性

<br>

**特点**

- 由于 Nginx 的高并发和静态资源缓存等特性，经常将静态资源部署在 Nginx 上

- 如果请求的是静态资源，直接到静态资源目录获取资源，如果是动态资源的请求，则利用反向代理的原理，把请求转发给对应后台应用去处理，从而实现动静分离

- 使用前后端分离后，可以很大程度提升静态资源的访问速度，即使动态服务不可用，静态资源的访问也不会受到影响

<br>

**配置**

```
# 处理静态资源
server {
    listen 80;

    location /static/ {
        # 使用 alias 的话路径后必须要加 / ==> /etc/nginx/static/
        alias  /静态资源存放路径/;
        autoindex on;
    }
}
```



<br>

### 正向代理

> 为客户端（用户）代理。意思是一个位于客户端和原始服务器(Origin Server)之间的服务器，为了从原始服务器取得内容，客户端向代理发送一个请求并指定目标(原始服务器)，然后代理向原始服务器转交请求并将获得的内容返回给客户端

<br>

**特点**

- 正向代理是为用户服务的，即为客户端服务的，客户端可以根据正向代理访问到它本身无法访问到的服务器资源
- 正向代理对用户是透明的，对服务端是非透明的，即服务端并不知道自己收到的是来自代理的访问还是来自真实客户端的访问

<br>

### 反向代理

> 为服务器代理，反向代理（Reverse Proxy）方式是指以代理服务器来接受网络上的连接请求，然后将请求转发给内部网络上的服务器，并将从服务器上得到的结果返回给网络上请求连接的客户端，此时代理服务器对外就表现为一个反向代理服务器

<br>

**特点**

- 反向代理是为服务端服务的，反向代理可以帮助服务器接收来自客户端的请求，帮助服务器做请求转发，负载均衡等
- 反向代理对服务端是透明的，对用户是非透明的，即用户并不知道自己访问的是代理服务器，而服务器知道反向代理在为他服务
- 优点
  1. 隐藏真实服务器
  2. 动静分离，提升系统性能
  3. 负载均衡，便于横向扩充后端动态服务

<br>

**参数**

**proxy_pass**，定义在 `location` 块中，用于配置代理服务器

- 语法为 `proxy_pass URL`

- `URL` 必须以 `http` 或 `https` 开头

- `URL` 中可以携带变量

- `URL` 中是否带 `URI` ，会直接影响发往上游请求的 `URL`

- 两种常见的 `URL` 用法

  1. `proxy_pass http://127.0.0.1:8080`

  2. `proxy_pass http://127.0.0.1:8080/`


  两种用法的区别就是带 `/` 和不带 `/` 

  - 不带 `/`，用户请求 `/bbs/abc/test.html`，请求到达上游应用服务器的 `/bbs/abc/test.html`
  - 带 `/` ，用户请求 `/bbs/abc/test.html`，请求到达上游应用服务器的 `/abc/test.html`

**配置**

```
# 反向代理设置
server {
    # 监听80端口
    listen 80;
    server_name localhost;

    # 代理路径 将对 localhost:80/ 的访问代理到 proxy_pass http://127.0.0.1:8082/
    # / 为基础配置，一般配置在其他location的最下方
    location / {
        proxy_pass http://127.0.0.1:8082;
    }
}

# 反向代理设置
server {
    listen 80;
    server_name www.nginx-test.com;

    # 匹配 www.nginx-test.com/8082/* 路径
    location /8082/ {
        proxy_pass http://127.0.0.1:8082;

        rewrite "^/8082/(.*)$" /$1 break;
    }
}
```

```
server {
    listen       80;
    server_name  sk-test.com;

    # 浏览器访问 http://sk-test.com/sk/** 实际访问 http://127.0.0.1:8888/sk/***
    location ~ /sk/ {
    proxy_pass http://127.0.0.1:8888;

    }
}
```



<br>

### 负载均衡

> 当⽤⼾访问⽹站的时候，先访问⼀个中间服务器，再让这个中间服务器根据定制好的负载均衡算法，在服务器集群中选择合适服务器，然后将该访问请求引⼊选择的服务器。⽤⼾每次访问都会保证服务器集群中的每个服务器压⼒趋于平衡，分担了服务器压⼒

<br>

**负载均衡策略**

1. 轮询策略
   
   默认情况下采用的策略，将所有客户端请求轮询分配给服务端
   
   如果其中某一台服务器压力太大，出现延迟，会影响所有分配在这台服务器下的用户
2. `ip_hash`
   
   来自同一个 IP 的请求永远只分配一台服务器。每个请求按请求 IP 的 hash 结果分配，每个 IP 都能固定访问一个后端的服务器，可以解决 Session 的问题。但 Session 问题不推荐使用 Nginx 解决，推荐 Redis
3. 最小连接数策略，将请求优先分配给压力较小的服务器，它可以平衡每个队列的长度，并避免向压力大的服务器添加更多的请求
4. 最快响应时间策略，优先分配给响应时间最短的服务器

<br>

**参数**

**upstream**，定义在 `http` 块中，用于定义上游服务器（指的就是后台提供的应用服务器）的相关信息

- `keepalive` 对上游服务启用长连接
- `keepalive_requests` 一个长连接最多请求 HTTP 的个数
- `keepalive_timeout` 空闲情形下，一个长连接的超时时长
- `hash` 哈希负载均衡算法
- `ip_hash` 依据 IP 进行哈希计算的负载均衡算法
- `least_conn` 最少连接数负载均衡算法
- `least_time` 最短响应时间负载均衡算法
- `random` 随机负载均衡算法
- `server` 定义上游服务器地址
  - `weight=number` 权重值，默认为1
  - `max_conns=number` 上游服务器的最大并发连接数
  - `backup` 备份服务器，仅当其他服务器都不可用时才会启用

**配置**

```conf
# 负载均衡设置一组针对 proxy_pass http://nginx-test.com 服务的负载均衡
upstream nginx-test.com {
  # 设置负载均衡机制，默认使用轮询机制
  # weight 配置每个服务器的权重，指定服务器轮询比例，weight 和访问比例成正比，用于后端服务器性能不均的情况
  server 127.0.0.1:8080 weight=1;
  server 127.0.0.1:8082 weight=1;
}

server {
  listen 80;
  server_name www.nginx-test.com;

  location / {
      proxy_pass http://nginx-test.com;
  }
}
```

<br>

### 缓存

> 开启缓存可以存储一些之前被访问过、而且可能将要被再次访问的资源，使用户可以直接从代理服务器获得，从而减少上游服务器的压力

<br>

**参数**

- **proxy_cache**，定义在 `location` 块中，存储一些之前被访问过，且可能会被再次访问的资源，可以直接从代理服务器获得，减少上游服务器的压力，加快整个访问速度
  - `proxy_cache zone | off `，zone 是共享内存的名称
  - 默认值 `off`
- proxy_cache_path，设置缓存文件的存放路径
- proxy_cache_key，设置缓存文件的 `key` 
- proxy_cache_valid，配置什么状态码可以被缓存，以及缓存时长

**配置**

```conf
server {
  listen 80;
  server_name localhost;
  location / {
    proxy_cache cache_zone; # 设置缓存内存
    proxy_cache_valid 200 5m; # 缓存状态为 200 的请求，缓存时长为 5 分钟
    proxy_cache_key $request_uri; # 缓存文件的 key 为请求的 URI
    add_header Nginx-Cache-Status $upstream_cache_status # 把缓存状态设置为头部信息，响应给客户端
    proxy_pass http://cache_server; # 代理转发
  }
}
```

<br>

### 限流

> Nginx的限流模块，是基于漏桶算法实现的，在⾼并发的场景下⾮常实⽤

<br>

**参数**

- **limit_req_zone**，定义在 `http` 块中，
- zone，`zone = ip状态:共享内存区域`，定义IP状态及URL访问频率的共享内存区域，`zone=keyword` 标识区域的名字，以及冒号后⾯跟区域⼤⼩
- rate，定义最⼤请求速率

**配置**

```conf
http {
    # $binary_remote_addr 表⽰保存客⼾端 IP 地址的⼆进制形式
    # mylimit zone=keyword 标识区域的名字
    # 10m 16000 个 IP 地址的状态信息约 1MB，所以⽰例中 10m 区域可以存储 160000 个 IP 地址
    # rate=100r/s 速率不能超过每秒 100 个请求
    limit_req_zone $binary_remote_addr zone=mylimit:10m rate=100r/s;
    
    server {
        location / {
            # burst 排队⼤⼩
            # nodelay 不限制单个请求间的时间
            limit_req zone = mylimit burst=20 nodelay;
        }
    }

}
```

<br>

### 黑白名单

> 可以拒绝某个（某组）IP访问

```conf
location / {
    deny 121.43.102.159;
    deny 121.84.165.0/24;
    deny all;
    
    allow 12.2.3.5;
}
```

<br>

### HTTPS

> 使用Nginx配置HTTPS支持

<br>

**HTTPS工作流程**

1. 客户端（浏览器）访问 `https://www.baidu.com` 百度网站；
2. 百度服务器返回 HTTPS 使用的 CA 证书；
3. 浏览器验证 CA 证书是否合法；
4. 验证通过，证书合法，生成一串随机数并使用公钥（证书中提供的）进行加密；
5. 发送公钥加密后的随机数给百度服务器；
6. 百度服务器拿到密文，通过私钥进行解密，获取到随机数（公钥加密，私钥解密，反之也可以）；
7. 百度服务器把要发送给浏览器的内容，使用随机数进行加密后传输给浏览器；
8. 此时浏览器可以使用公钥进行解密，获取到服务器的真实传输内容；

这就是 HTTPS 的基本运作原理，对称加密和非对称加密配合使用，保证传输内容的安全性

[HTTPS 双向认证原理](http://www.884358.com/https-auth/)

**配置证书**

下载 HTTPS 证书的压缩文件，里面有个 Nginx 文件夹，把 `xxx.crt` 和 `xxx.key` 文件拷贝到服务器目录，再进行配置

```conf
server {
  listen 443 ssl http2 default_server;   # SSL 访问端口号为 443
  server_name nginx-test.com;         # 填写绑定证书的域名
  ssl_certificate /etc/nginx/https/nginx-test.com_bundle.crt;   # 证书地址
  ssl_certificate_key /etc/nginx/https/nginx-test.com.key;      # 私钥地址
  ssl_session_timeout 10m;
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # 支持 ssl 协议版本，默认为后三个，主流版本是[TLSv1.2]
 
  location / {
    root         /usr/share/nginx/html;
    index        index.html index.htm;
  }
}
```

<br>

### 跨域访问

> 跨域，通过不同 IP 或端口进行资源访问



**配置**

如

- 前端服务为 `fronten.server.com`，后端服务为 `backen.server.com`

```conf
server {
    listen 80;
    server_name  fronten.server.com;
    
    location / {
        proxy_pass backen.server.com;
    }
}
```

