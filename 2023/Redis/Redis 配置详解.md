# Redis 配置详解



## 前言

> Redis 配置文件会发现一个词 “ASAP” 经常出现在文件中，是 "As Soon As Possible" 的缩写



## 线程与 IO

> Redis 基本上可以说是单线程运行的，除了某些如 `UNLINK` 以及慢 IO 访问等需要其他需要多线程访问的情况



>Redis 目前可以使用不同的读写线程来进行读写操作。由于写入操作比较慢，Redis 通常使用管道以加速每个核心的写入性能，并且面对大量的写入场景会启用多个线程实例。使用多个读写线程可以不借助管道或者实例分片，将 Redis 的读写速率提高 2 倍
>



<br>

## 数据持久化

### RDB

#### 同步策略

>Save the DB to disk.
>
>`save <seconds> <changes> [<seconds> <changes> ...]`
>
>
>
>Redis will save the DB if the given number of seconds elapsed and it surpassed the given number of write operations against the DB.
>
>Snapshotting can be completely disabled with a single empty string argument
>
>as in following example:
>
>`save ""`
>
>
>
>Unless specified otherwise, by default Redis will save the DB:
>
>* After 3600 seconds (an hour) if at least 1 change was performed
>* After 300 seconds (5 minutes) if at least 100 changes were performed
>* After 60 seconds if at least 10000 changes were performed
>
>You can set these explicitly by uncommenting the following line.
>
>`save 3600 1 300 100 60 10000`



#### 名称与路径

>The filename where to dump the DB
>
>`dbfilename dump.rdb`
>
>
>
>The working directory. 
>
>The DB will be written inside this directory, with the filename specified above using the 'dbfilename' configuration directive. The Append Only File will also be created inside this directory.
>
>Note that you must specify a directory here, not a file name.
>
>`dir /usr/local/var/db/redis/`





#### 错误恢复

```conf
# 默认情况下，如果开启了 RDB 快照，且最近一次 RDB 后台保存失败，就会停止写操作
#
# 这会以一种强硬的方式让用户知道，数据没能正确的在磁盘上进行持久化
#
# 如果后台执行保存工作的进程重新运行，Redis 会重新自动允许写操作发生
#
# 如果设置了自定义的监视器来监视 Redis 服务器的持久化，将此功能关闭后，即使
# 磁盘写入出现问题 Redis 也会照常工作
stop-writes-on-bgsave-error yes
```



#### 压缩与校验

```
# 启用 LZF 压缩算法
# By default compression is enabled as it's almost always a win.
# If you want to save some CPU in the saving child set it to 'no' but
# the dataset will likely be bigger if you have compressible values or keys.
rdbcompression yes

# RDB 文件校验码，一般出现在 RDB 文件的尾部
# Since version 5 of RDB a CRC64 checksum is placed at the end of the file.
# This makes the format more resistant to corruption but there is a performance
# hit to pay (around 10%) when saving and loading RDB files, so you can disable it
# for maximum performances.
# 在保证文件完整性和准确性上有很大保证，但是在保存 RDB 文件的时候会多大约 10% 的性能损耗
# 可将其禁用以获得更的高性能
#
# 将校验关闭后，RDB 文件尾部会使用 0 来作为标识，加载到拥有 0 标识的文件时会跳过校验
rdbchecksum yes
```



### AOF

>使用异步 RDB 的方式将 dump 文件保存到磁盘上这种方法在很多情况下已经够用了，但是 Redis 进程或者服务器断电等问题会导致存在几分钟的数据丢失
>
>
>
>AOF 可以提供一个更加完善的持久化方式，对于默认使用 `fsync` 策略的 Redis 服务器，在遇到服务器断电或者单个写程序线程在遇到问题时，仅会丢失几秒钟的数据
>
>
>
>AOF 和 RDB 持久化可同时开启，在 Redis 启动时，如果同时存在 AOF 和 RDB 数据，会优先选择使用 AOF 持久化的数据来恢复，因为 AOF 具有更好的数据持久化保证

在 Redis 中配置  `appendonly <yes|no>` 来修改 AOF 状态。



#### 同步策略

当 Redis 开启 AOF 持久化时，Redis 会将所有的写命令以日志的形式追加到 AOF 文件中。当 AOF 文件过大时，Redis 会启动 AOF 重写操作，以此来压缩 AOF 文件。AOF 重写的过程是将内存中的数据写入到新的 AOF 文件中。在这个过程中，Redis 可以采用两种同步策略：

* 每次写入 AOF 文件时，都进行一次 fsync 操作，将缓冲区的数据刷入磁盘。这种方式会保证 AOF 文件的持久化和安全性，但是会导致写入性能下降。
* Redis 从上次 fsync 操作以来，累计写入一定数量的数据后，才进行一次 fsync 操作。这种方式会提高写入性能，但是会带来数据安全性的风险。因为在 fsync 之前，数据仅停留在缓冲区中，如果系统出现宕机等异常情况，那么这部分数据就会丢失。



> 调用 `fsync()` （Append-only file sync）会让操作系统将 AOF 文件直接写到磁盘上，无需等缓存中有足够 AOF 文件数据再转存至磁盘（不同的操作系统会有不同的响应，有些操作系统直接写入磁盘，而有些操作系统只是尝试尽量这样做）
>
> `appendfsync` 选项决定了写命令被追加到 AOF 文件后是否需要立即同步到磁盘：
>
> * no 不会执行同步操作，让系统决定何时将 AOF 文件同步到磁盘
> * always 每次写命令被追加到 AOF 文件后都会立即同步到磁盘
> * everysec 每秒钟会执行一次同步操作，默认配置
>
> 
>
> `#appendfsync always`
>
> `appendfsync everysec`
>
> `#appendfsync no`



> 当 AOF 文件同步策略设置为 *always* 或 *everysec*，且存在一个后台进程正在执行大量的磁盘 IO 操作，在某些 Linux 上 Redis 可能会在调用 `fsync` 时阻塞很长的时间。目前还没有办法修复。
>
> 
>
> 为了缓解这个问题，可以启用 `no-appendfsync-on-rewrite` 阻止主线程在执行 `BGSAVE` 或 `BGREWRITEAOF` 的时候调用 `fsync`
>
> 
>
> 当存在子线程进行保存备份操作的时候 Redis 的 AOF 文件同步策略相当于 `appendfsync no`，也就是说此时可能会丢失大概 30s 左右时间的数据
>
> `no-appendfsync-on-rewrite no`



#### 基准文件创建

> Redis can create append-only base files in either RDB or AOF formats. Using the RDB format is always faster and more efficient, and disabling it is only supported for backward compatibility purposes. 默认使用 RDB 文件作为基准创建 AOF 文件
>
> `aof-use-rdb-preamble yes`
>
> 
>
> Redis supports recording timestamp annotations in the AOF to support restoring the data from a specific point-in-time. However, using this capability changes the AOF format in a way that may not be compatible with existing AOF parsers. 是否给 AOF 记录加时间戳，可能存在兼容性问题
>
> `aof-timestamp-enabled no`



#### 名称与路径

**文件类型**

> Redis 7 and newer use a set of append-only files to persist the dataset and changes applied to it. Redis 7 之后使用一系列文件来进行数据持久化以及应用数据更改。
>
> - Base files, which are a snapshot representing the complete state of the dataset at the time the file was created. Base files can be either in the form of RDB (binary serialized) or AOF (textual commands).
>
>   基础文件，表示快照文件创建时整个数据库中数据的状态，基础文件可以是 RDB 形式的文件，也可以是 AOF 文件
>
> - Incremental files, which contain additional commands that were applied to the dataset following the previous file.
>
>   增量文件，包含对基础文件之后针对数据集操作的附加命令



**文件名称**

> Append-only file names are created by Redis following a specific pattern. The file name's prefix is based on the 'appendfilename' configuration parameter, followed by additional information about the sequence and type.
>
> `appendfilename "appendonly.aof"`
>
> For example, if appendfilename is set to appendonly.aof, the following file names could be derived:
>
> ---- appendonly.aof.1.base.rdb as a base file.
>
> ----- appendonly.aof.1.incr.aof, appendonly.aof.2.incr.aof as incremental files.
>
> ---- appendonly.aof.manifest as a manifest file.



**文件保存路径**

> For convenience, Redis stores all persistent append-only files in a dedicated directory. The name of the directory is determined by the appenddirname configuration parameter.
>
> `appenddirname "appendonlydir"`



#### 文件重写

> Automatic rewrite of the append only file. Redis is able to automatically rewrite the log file implicitly calling BGREWRITEAOF when the AOF log size grows by the specified percentage.
>
> 在 AOF 文件大小增长到指定范围时，Redis 能够通过隐式调用 BGREWRITEAOF 来重写 AOF 文件。
>
> Redis remembers the size of the AOF file after the latest rewrite (if no rewrite has happened since the restart, the size of the AOF at startup is used).
>
> This base size is compared to the current size. If the current size is bigger than the specified percentage, the rewrite is triggered. Also you need to specify a minimal size for the AOF file to be rewritten, this is useful to avoid rewriting the AOF file even if the percentage increase is reached but it is still pretty small.
>
> Redis 会使用最新一次重写时 AOF 文件的大小来作为基准，如果还没进行过重写操作，则使用 Redis 启动时 AOF 文件的大小来作为基准。
>
> 用当前文件大于来和基准大小做比较，当前文件大小超过指定阈值，且大于基准大小乘上对应百分比，就会触发重写操作。默认是当 AOF 文件大于 64m 且大于上一次重写文件的 2 倍触发重写
>
> Specify a percentage of zero in order to disable the automatic AOF rewrite feature.
>
> `auto-aof-rewrite-percentage 100`
> `auto-aof-rewrite-min-size 64mb`



> When a child rewrites the AOF file, if the following option is enabled the file will be fsync-ed every 4 MB of data generated. This is useful in order to commit the file to the disk more incrementally and avoid big latency spikes. 
>
> `aof-rewrite-incremental-fsync yes`



#### 文件校验

> An AOF file may be found to be truncated at the end during the Redis startup process, when the AOF data gets loaded back into memory. This may happen when the system where Redis is running crashes, especially when an ext4 filesystem is mounted without the data=ordered option (however this can't happen when Redis itself crashes or aborts but the operating system still works correctly).
>
> Redis can either exit with an error when this happens, or load as much data as possible (the default now) and start if the AOF file is found to be truncated at the end. The following option controls this behavior.
>
> If aof-load-truncated is set to yes, a truncated AOF file is loaded and the Redis server starts emitting a log to inform the user of the event. Otherwise if the option is set to no, the server aborts with an error and refuses to start. When the option is set to no, the user requires to fix the AOF file using the "redis-check-aof" utility before to restart the server.
>
> Note that if the AOF file will be found to be corrupted in the middle, the server will still exit with an error. This option only applies when Redis will try to read more data from the AOF file but not enough bytes will be found.
>
> `aof-load-truncated yes`
>
> 
>
> Redis 启动过程中，将 AOF 数据加载回内存时，可能会发现 AOF 文件在末尾被截断。Redis 可能会因此退出，也可能继续加载数据并启动。
>
> 如果 `aof-load-truncated` 设置为 *yes*，Redis 会发送相关日志通知用户此事件；否则 Redis 会出错并拒绝启动。当设置为 *no* 时，需要使用 `redis-check-aof` 命令修复 AOF 文件才能重新启动；如果在加载 AOF 文件时发现文件损坏，Redis 服务会报错并退出。



<br>

## 内存管理

### 内存回收策略

Redis 设置最大内存容量 `maxmemory <bytes>`



> 当达到内存限制时，Redis 将会根据缓存驱逐策略删除 Key



**POLICY**:

* volatile-lru（Least Recently Used） -> 从**已设置过期**时间数据中淘汰**最近最少使用**数据
* allkeys-lru -> 从**所有**数据中淘汰**最近最少使用**数据
* volatile-lfu（Least Frequently Used） -> 从**已设置过期**时间数据中淘汰**最不经常使用**数据
* allkeys-lfu -> 从**所有**数据中淘汰**最不经常使用**数据
* volatile-random -> 从**已设置过期**时间数据中**随机**淘汰数据
* allkeys-random -> 从**所有**时间数据中**随机**淘汰数据
* volatile-ttl -> 从**已设置过期**时间数据中移除将要过期数据
* noeviction -> 不淘汰数据，写入操作报错



> 如果 Redis 不能根据缓存策略删除 Key 或者缓存删除策略是 `noeviction`：当内存满时 Redis 将会给 SET/LPUSH 等新增方法返回错误的结果，但数据读取方法依然正常



> Both LRU, LFU and volatile-ttl are implemented using approximated randomized algorithms(in order to save memory). 



> By default Redis will check five keys and pick the one that was used least recently, you can change the sample size using the following configuration directive.
>
> 默认一次检查 5 个 key，从其中一个选出满足策略的 key 并将其淘汰
>
> The default of 5 produces good enough results. 10 Approximates very closely true LRU but costs more CPU. 3 is faster but not very accurate.



### 过期数据回收策略

* **惰性删除**，在访问到过期 key 时将其删除；
* **主动回收**，在后台缓慢扫描寻找过期的 key，就可以释放过期/不会再被访问到的 key 占用的空间。



> Redis 的默认策略会尽量避免内存中存在超过 10% 的过期 Key，并尽量避免过期的 Key 消耗超过总内存 25% 的空间



### Key 删除原语

Redis 中有两个删除 key 的原语：

* DEL

  > DEL is a blocking deletion of the object. It means that the server stops processing new commands in order to reclaim all the memory associated with an object in a synchronous way.

  > If the key deleted is associated with a small object, the time needed in order to execute the DEL command is very small and comparable to most other O(1) or O(log_N) commands in Redis.  However if the key is associated with an aggregated value containing millions of elements, the server can block for a long time (even seconds) in order to complete the operation.

* UNLINK

  > For the above reasons Redis also offers non blocking deletion primitives UNLINK (non blocking DEL) and the ASYNC option of FLUSHALL and FLUSHDB commands, in order to reclaim memory in background. 
  >
  > Those commands are executed in constant time. Another thread will incrementally free the object in the background as fast as possible.
  >
  > 这些命令由一个线程在一定的时间内执行，另一个线程在后台尽可能快的释放内存



> DEL, UNLINK and ASYNC option of FLUSHALL and FLUSHDB are user-controlled. It's up to the design of the application to understand when it is a good idea to use one or the other. 
>
> 以上删除命令是供用户使用的，由应用开发者来决定使用何种方式



> However the Redis server sometimes has to delete keys or flush the whole database as a side effect of other operations. 
>
> Redis 服务器有些时候也会主动的删除或者清空整个数据库数据，特别是以下这些场景：
>
> Specifically Redis deletes objects independently of a user call in the following scenarios:
>
> 1) On eviction, because of the maxmemory and maxmemory policy configurations, in order to make room for new data, without going over the specified memory limit. 
>
>    达到最大内存限制
>
> 2) Because of expire: when a key with an associated time to live must be deleted from memory.
>
>    当 key 过期
>
> 3) Because of a side effect of a command that stores data on a key that may already exist. For example the RENAME command may delete the old key content when it is replaced with another one. Similarly SUNIONSTORE or SORT with STORE option may delete existing keys. The SET command itself removes any old content of the specified key in order to replace it with the specified string.
>
>    执行相关的命令可能会将以存在的数据删除，如 RENAME 命令会将旧的键值删除；类似的还有 SUNIONSTORE 和 带有 STORE 参数的 SORT 命令也会删除已存在的 key；常用的 SET 命令本身也会将旧的内容（如果存在）删除，并替换成新的内容
>
> 4) During replication, when a replica performs a full resynchronization with its master, the content of the whole database is removed in order to load the RDB file just transferred.
>
>    从库从主库接收到一个完整的 resynchronization 请求时，整个从库的数据都会被移除，然后重新加载刚传输过来的 RDB 文件数据
>
> 
>
> In all the above cases the default is to delete objects in a blocking way, like if DEL was called. However you can configure each case specifically in order to instead release memory in a non-blocking way like if UNLINK was called, using the following configuration directives. 以上删除 Key 的方式都是阻塞进行的



<br>

## 高级配置

### Rehash

>Active rehashing uses 1 millisecond every 100 milliseconds of CPU time in order to help rehashing the main Redis hash table (the one mapping top-level keys to values). The hash table implementation Redis uses (see dict.c) performs a lazy rehashing: the more operation you run into a hash table that is rehashing, the more rehashing "steps" are performed, so if the server is idle the rehashing is never complete and some more memory is used by the hash table.
>
>`activerehashing yes`


<br>

## 内存碎片整理



## 主从配置

```
# Master-Replica replication. Use replicaof to make a Redis instance a copy of
# another Redis server. A few things to understand ASAP about Redis replication.
#
#   +------------------+      +---------------+
#   |      Master      | ---> |    Replica    |
#   | (receive writes) |      |  (exact copy) |
#   +------------------+      +---------------+
#
# 1) Redis replication is asynchronous, but you can configure a master to
#    stop accepting writes if it appears to be not connected with at least
#    a given number of replicas.
# 2) Redis replicas are able to perform a partial resynchronization with the
#    master if the replication link is lost for a relatively small amount of
#    time. You may want to configure the replication backlog size (see the next
#    sections of this file) with a sensible value depending on your needs.
# 3) Replication is automatic and does not need user intervention. After a
#    network partition replicas automatically try to reconnect to masters
#    and resynchronize with them.
#
# replicaof <masterip> <masterport>

# If the master is password protected (using the "requirepass" configuration
# directive below) it is possible to tell the replica to authenticate before
# starting the replication synchronization process, otherwise the master will
# refuse the replica request.
#
# masterauth <master-password>
#
# However this is not enough if you are using Redis ACLs (for Redis version
# 6 or greater), and the default user is not capable of running the PSYNC
# command and/or other commands needed for replication. In this case it's
# better to configure a special user to use with replication, and specify the
# masteruser configuration as such:
#
# masteruser <username>
#
# When masteruser is specified, the replica will authenticate against its
# master using the new AUTH form: AUTH <username> <password>.

# When a replica loses its connection with the master, or when the replication
# is still in progress, the replica can act in two different ways:
#
# 1) if replica-serve-stale-data is set to 'yes' (the default) the replica will
#    still reply to client requests, possibly with out of date data, or the
#    data set may just be empty if this is the first synchronization.
#
# 2) If replica-serve-stale-data is set to 'no' the replica will reply with error
#    "MASTERDOWN Link with MASTER is down and replica-serve-stale-data is set to 'no'"
#    to all data access commands, excluding commands such as:
#    INFO, REPLICAOF, AUTH, SHUTDOWN, REPLCONF, ROLE, CONFIG, SUBSCRIBE,
#    UNSUBSCRIBE, PSUBSCRIBE, PUNSUBSCRIBE, PUBLISH, PUBSUB, COMMAND, POST,
#    HOST and LATENCY.
#
replica-serve-stale-data yes

# You can configure a replica instance to accept writes or not. Writing against
# a replica instance may be useful to store some ephemeral data (because data
# written on a replica will be easily deleted after resync with the master) but
# may also cause problems if clients are writing to it because of a
# misconfiguration.
#
# Since Redis 2.6 by default replicas are read-only.
#
# Note: read only replicas are not designed to be exposed to untrusted clients
# on the internet. It's just a protection layer against misuse of the instance.
# Still a read only replica exports by default all the administrative commands
# such as CONFIG, DEBUG, and so forth. To a limited extent you can improve
# security of read only replicas using 'rename-command' to shadow all the
# administrative / dangerous commands.
replica-read-only yes

# Replication SYNC strategy: disk or socket.
#
# New replicas and reconnecting replicas that are not able to continue the
# replication process just receiving differences, need to do what is called a
# "full synchronization". An RDB file is transmitted from the master to the
# replicas.
#
# The transmission can happen in two different ways:
#
# 1) Disk-backed: The Redis master creates a new process that writes the RDB
#                 file on disk. Later the file is transferred by the parent
#                 process to the replicas incrementally.
# 2) Diskless: The Redis master creates a new process that directly writes the
#              RDB file to replica sockets, without touching the disk at all.
#
# With disk-backed replication, while the RDB file is generated, more replicas
# can be queued and served with the RDB file as soon as the current child
# producing the RDB file finishes its work. With diskless replication instead
# once the transfer starts, new replicas arriving will be queued and a new
# transfer will start when the current one terminates.
#
# When diskless replication is used, the master waits a configurable amount of
# time (in seconds) before starting the transfer in the hope that multiple
# replicas will arrive and the transfer can be parallelized.
#
# With slow disks and fast (large bandwidth) networks, diskless replication
# works better.
repl-diskless-sync yes

# When diskless replication is enabled, it is possible to configure the delay
# the server waits in order to spawn the child that transfers the RDB via socket
# to the replicas.
#
# This is important since once the transfer starts, it is not possible to serve
# new replicas arriving, that will be queued for the next RDB transfer, so the
# server waits a delay in order to let more replicas arrive.
#
# The delay is specified in seconds, and by default is 5 seconds. To disable
# it entirely just set it to 0 seconds and the transfer will start ASAP.
repl-diskless-sync-delay 5

# When diskless replication is enabled with a delay, it is possible to let
# the replication start before the maximum delay is reached if the maximum
# number of replicas expected have connected. Default of 0 means that the
# maximum is not defined and Redis will wait the full delay.
repl-diskless-sync-max-replicas 0

# -----------------------------------------------------------------------------
# WARNING: RDB diskless load is experimental. Since in this setup the replica
# does not immediately store an RDB on disk, it may cause data loss during
# failovers. RDB diskless load + Redis modules not handling I/O reads may also
# cause Redis to abort in case of I/O errors during the initial synchronization
# stage with the master. Use only if you know what you are doing.
# -----------------------------------------------------------------------------
#
# Replica can load the RDB it reads from the replication link directly from the
# socket, or store the RDB to a file and read that file after it was completely
# received from the master.
#
# In many cases the disk is slower than the network, and storing and loading
# the RDB file may increase replication time (and even increase the master's
# Copy on Write memory and replica buffers).
# However, parsing the RDB file directly from the socket may mean that we have
# to flush the contents of the current database before the full rdb was
# received. For this reason we have the following options:
#
# "disabled"    - Don't use diskless load (store the rdb file to the disk first)
# "on-empty-db" - Use diskless load only when it is completely safe.
# "swapdb"      - Keep current db contents in RAM while parsing the data directly
#                 from the socket. Replicas in this mode can keep serving current
#                 data set while replication is in progress, except for cases where
#                 they can't recognize master as having a data set from same
#                 replication history.
#                 Note that this requires sufficient memory, if you don't have it,
#                 you risk an OOM kill.
repl-diskless-load disabled

# Master send PINGs to its replicas in a predefined interval. It's possible to
# change this interval with the repl_ping_replica_period option. The default
# value is 10 seconds.
#
# repl-ping-replica-period 10

# The following option sets the replication timeout for:
#
# 1) Bulk transfer I/O during SYNC, from the point of view of replica.
# 2) Master timeout from the point of view of replicas (data, pings).
# 3) Replica timeout from the point of view of masters (REPLCONF ACK pings).
#
# It is important to make sure that this value is greater than the value
# specified for repl-ping-replica-period otherwise a timeout will be detected
# every time there is low traffic between the master and the replica. The default
# value is 60 seconds.
#
# repl-timeout 60

# Disable TCP_NODELAY on the replica socket after SYNC?
#
# If you select "yes" Redis will use a smaller number of TCP packets and
# less bandwidth to send data to replicas. But this can add a delay for
# the data to appear on the replica side, up to 40 milliseconds with
# Linux kernels using a default configuration.
#
# If you select "no" the delay for data to appear on the replica side will
# be reduced but more bandwidth will be used for replication.
#
# By default we optimize for low latency, but in very high traffic conditions
# or when the master and replicas are many hops away, turning this to "yes" may
# be a good idea.
repl-disable-tcp-nodelay no

# Set the replication backlog size. The backlog is a buffer that accumulates
# replica data when replicas are disconnected for some time, so that when a
# replica wants to reconnect again, often a full resync is not needed, but a
# partial resync is enough, just passing the portion of data the replica
# missed while disconnected.
#
# The bigger the replication backlog, the longer the replica can endure the
# disconnect and later be able to perform a partial resynchronization.
#
# The backlog is only allocated if there is at least one replica connected.
#
# repl-backlog-size 1mb

# After a master has no connected replicas for some time, the backlog will be
# freed. The following option configures the amount of seconds that need to
# elapse, starting from the time the last replica disconnected, for the backlog
# buffer to be freed.
#
# Note that replicas never free the backlog for timeout, since they may be
# promoted to masters later, and should be able to correctly "partially
# resynchronize" with other replicas: hence they should always accumulate backlog.
#
# A value of 0 means to never release the backlog.
#
# repl-backlog-ttl 3600

# The replica priority is an integer number published by Redis in the INFO
# output. It is used by Redis Sentinel in order to select a replica to promote
# into a master if the master is no longer working correctly.
#
# A replica with a low priority number is considered better for promotion, so
# for instance if there are three replicas with priority 10, 100, 25 Sentinel
# will pick the one with priority 10, that is the lowest.
#
# However a special priority of 0 marks the replica as not able to perform the
# role of master, so a replica with priority of 0 will never be selected by
# Redis Sentinel for promotion.
#
# By default the priority is 100.
replica-priority 100

# The propagation error behavior controls how Redis will behave when it is
# unable to handle a command being processed in the replication stream from a master
# or processed while reading from an AOF file. Errors that occur during propagation
# are unexpected, and can cause data inconsistency. However, there are edge cases
# in earlier versions of Redis where it was possible for the server to replicate or persist
# commands that would fail on future versions. For this reason the default behavior
# is to ignore such errors and continue processing commands.
#
# If an application wants to ensure there is no data divergence, this configuration
# should be set to 'panic' instead. The value can also be set to 'panic-on-replicas'
# to only panic when a replica encounters an error on the replication stream. One of
# these two panic values will become the default value in the future once there are
# sufficient safety mechanisms in place to prevent false positive crashes.
#
# propagation-error-behavior ignore

# Replica ignore disk write errors controls the behavior of a replica when it is
# unable to persist a write command received from its master to disk. By default,
# this configuration is set to 'no' and will crash the replica in this condition.
# It is not recommended to change this default, however in order to be compatible
# with older versions of Redis this config can be toggled to 'yes' which will just
# log a warning and execute the write command it got from the master.
#
# replica-ignore-disk-write-errors no

# -----------------------------------------------------------------------------
# By default, Redis Sentinel includes all replicas in its reports. A replica
# can be excluded from Redis Sentinel's announcements. An unannounced replica
# will be ignored by the 'sentinel replicas <master>' command and won't be
# exposed to Redis Sentinel's clients.
#
# This option does not change the behavior of replica-priority. Even with
# replica-announced set to 'no', the replica can be promoted to master. To
# prevent this behavior, set replica-priority to 0.
#
# replica-announced yes

# It is possible for a master to stop accepting writes if there are less than
# N replicas connected, having a lag less or equal than M seconds.
#
# The N replicas need to be in "online" state.
#
# The lag in seconds, that must be <= the specified value, is calculated from
# the last ping received from the replica, that is usually sent every second.
#
# This option does not GUARANTEE that N replicas will accept the write, but
# will limit the window of exposure for lost writes in case not enough replicas
# are available, to the specified number of seconds.
#
# For example to require at least 3 replicas with a lag <= 10 seconds use:
#
# min-replicas-to-write 3
# min-replicas-max-lag 10
#
# Setting one or the other to 0 disables the feature.
#
# By default min-replicas-to-write is set to 0 (feature disabled) and
# min-replicas-max-lag is set to 10.

# A Redis master is able to list the address and port of the attached
# replicas in different ways. For example the "INFO replication" section
# offers this information, which is used, among other tools, by
# Redis Sentinel in order to discover replica instances.
# Another place where this info is available is in the output of the
# "ROLE" command of a master.
#
# The listed IP address and port normally reported by a replica is
# obtained in the following way:
#
#   IP: The address is auto detected by checking the peer address
#   of the socket used by the replica to connect with the master.
#
#   Port: The port is communicated by the replica during the replication
#   handshake, and is normally the port that the replica is using to
#   listen for connections.
#
# However when port forwarding or Network Address Translation (NAT) is
# used, the replica may actually be reachable via different IP and port
# pairs. The following two options can be used by a replica in order to
# report to its master a specific set of IP and port, so that both INFO
# and ROLE will report those values.
#
# There is no need to use both the options if you need to override just
# the port or the IP address.
#
# replica-announce-ip 5.5.5.5
# replica-announce-port 1234
```



## 集群配置



<br>

## 参考

[Redis#persistence](https://redis.io/topics/persistence)

[redis-persistence-demystified](http://antirez.com/post/redis-persistence-demystified.html)