# Redis 数据类型



## Key

>支持使用二进制和空字符串来作为 Key，但是空字符串 Key 只能有且只有一个

> A few other rules about keys:
>
> * 太长的 Key 不推荐，不仅消耗更多的内存空间，进行键值比较的消费也更大。可以通过对键值进行哈希操作来解决
>
> * 太短的 Key 不推荐，可读性不高，但是短的 Key 可以带来更小的消耗，这需要在开发中进行权衡
>
> * Try to stick with a schema. 推荐给 Key 定制约束，比如 "user:1000"，"comment:4321:reply.to" ，"comment:4321:reply-to" 等，可以使用 `.` 或 `-` 来分割复杂的字段
>
> * The maximum allowed key size is 512 MB. 键值允许的最大 size 为 512MB



### 操作

```shell
EXISTS
DEL
TYPE

SET
MSET # 批量操作

GET
MGET # 批量操作
```



### 过期时间

> Key expiration lets you set a timeout for a key, also known as a "time to live", or "TTL". When the time to live elapses, the key is automatically destroyed.
>
> 
>
> A few important notes about key expiration:
>
> - They can be set both using seconds or milliseconds precision.
> - However the expire time resolution is always 1 millisecond. 时间精度为 1 毫秒
> - Information about expires are replicated and persisted on disk, the time virtually passes when your Redis server remains stopped (this means that Redis saves the date at which a key will expire). 过期时间会备份并持久化到磁盘上，即使 Redis 服务器出错，也能保证设置有过期时间的数据信息完整
>
> 
>
> `expire key seconds [NX | XX | GT | LT]`
>
> `PEXPIRE key milliseconds [NX | XX | GT | LT]`



> `TTL` Returns the remaining time to live of a key that has a timeout.
>
> `TTL key`



> `PTTL` returns it in milliseconds.
>
> `PTTL key` 



### 持久化

> Remove the existing timeout on `key`, turning the key from *volatile* (a key with an expire set) to *persistent* (a key that will never expire as no timeout is associated).
>
> `PERSIST key`



## String



### 原子操作

`INCR`

> Increments the number stored at `key` by one. INCR is atomic, even multiple clients issuing INCR against the same key will never enter into a race condition
>
> 
>
> If the key does not exist, it is set to `0` before performing the operation. An error is returned if the key contains a value of the wrong type or contains a string that can not be represented as integer. This operation is limited to 64 bit signed integers.
>
> 
>
> **Note**: this is a string operation because Redis does not have a dedicated integer type.
>
> `INCR key`
>
> 
>
> Usage: Counter and Rate Limiter.
>
> ```lua
> -- Rate Limiter
> FUNCTION LIMIT_API_CALL(ip)
> ts = CURRENT_UNIX_TIME()
> keyname = ip+":"+ts
> MULTI
>     INCR(keyname)
>     EXPIRE(keyname,10)
> EXEC
> current = RESPONSE_OF_INCR_WITHIN_MULTI
> IF current > 10 THEN
>     ERROR "too many requests per second"
> ELSE
>     PERFORM_API_CALL()
> END
> ```



> 类似的还有 `DECR`



`INCRBY`

> Increments the number stored at `key` by `increment`. If the key does not exist, it is set to `0` before performing the operation. An error is returned if the key contains a value of the wrong type or contains a string that can not be represented as integer. 
>
> `INCRBY key increment`
>
> 



> 类似的还有 `DECRBY`



## List

> Redis lists are linked lists of string values. Redis lists are frequently used to:
>
> - Implement stacks and queues.
> - Build queue management for background worker systems.



## ZSet

>Sorted sets are used to store ordered collections of data with associated scores



## HyperLogLog

> HyperLogLog 是一个可以估算集合基数的数据结构，作为一种概率数据结构，HyperLogLog 以准确性换取高效的空间利用。

> The Redis HyperLogLog implementation uses up to 12 KB and provides a standard error of 0.81%.



### 样例

```shell
# PFADD dailyCount:{date} userId
> PFADD weekCount:{userId} weekday
> PFADD weekCount:001 1
> PFADD weekCount:001 2
> PFADD weekCount:001 3

> PFCOUNT weekCount:001
(integer) 3
```



> [HyperLogLog 使用場景](https://www.readfog.com/a/1664611911653756928)



## Bitmaps

>Bitmaps are used to store bit-level data
>
>Redis bitmaps are an extension of the string data type that lets you treat a string like a bit vector. You can also perform bitwise operations on one or more strings. Some examples of bitmap use cases include:
>
>- Efficient set representations for cases where the members of a set correspond to the integers 0-N.
>- Object permissions, where each bit represents a particular permission, similar to the way that file systems store permissions.



> 如何区分 HyperLogLog 和 Bitmap 的使用场景？
>
> HyperLogLog 可以统计用户对网站的访问情况，Bitmap 可以记录用户的签到等状态。看起来有些相似，但是可以这样子看：HyperLogLog 主要用来去重和统计信息，针对某一个数据进行统计；而 Bitmap 主要用来记录用户的行为信息，针对某一个操作进行记录。
