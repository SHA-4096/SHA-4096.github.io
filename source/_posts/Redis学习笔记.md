---
title: Redis学习笔记
mathjax: true
date: 2024/4/17
---


# Redis学习笔记

## 启动Redis

```bash
redis-cli#进入交互模式
```

## redis数据类型



### Keys

任何的二进制数据都可以作为redis的key，比如图片之类的，但是大小最大只能有512MB

官方推荐的规范命名法

- object-type:id 是官方推荐的命名方法（大概意思是用:分割层次结构）

  ```
  user:1000
  ```

  

- 当object-type或者id由多个字符组成的时候，建议使用"."或者"-"分割

  ```
  comment:4321:reply.to
  ```

### Strings

```
set [key] [value] #设置键值对
setnx [key] [value] #当key不存在的时候设置键值对
get [key] #获取键对应的值
mget [key...] #取得多个键值对
incr [key]  #值自增（对于整数）
incrby [key] [increment] #增加特定的值
```

### Key Expiration

设置key的TTL(Time to live) ，当TTL到期的时候，key就会被自动销毁	

有几点注意的

- Redis的过期时间分辨率是ms
- 即使server没有运行，计时也不会停止（Redis存储的是过期的日期）

#### 相关命令

```
expire [key] [seconds] #设置过期时间
persist [key] #撤销过期时间
ttl [key] #查看过期时间(i.e.还有多久过期)
pexpire [key] [ms] #ms为单位设置过期时间
pttl [key]#ms为单位查看过期时间
```

### Lists

概念问题：List != Array

> To explain the List data type it's better to start with a little bit of theory, as the term *List* is often used in an improper way by information technology folks. For instance "Python Lists" are not what the name may suggest (Linked Lists), but rather Arrays (the same data type is called Array in Ruby actually).

在Redis中，List指的是链表(Linked List)，插入元素复杂度是线性的

```bash
LPUSH #在头部插入元素（插入后返回元素个数）
RPUSH #在尾部插入元素
LRANGE key start stop#查看链表在[start,stop]的元素（下标从0开始）（下标为负数的时候表示从尾部数起）
LPOP #头部弹出（没有元素的时候返回nil）
RPOP #尾部弹出
LTRIM key start stop #读取指定范围的元素并且只保留它们

```

#### 使用技巧

LTRIM和LPUSH配合使用，可以实现滚动更新列表元素

```
LPUSH mylist <some element>
LTRIM mylist 0 999
```

### Blocking Operations On Lists

假设有A、B两个线程，A向列表里面塞数据，B从列表读取数据，为了避免B在列表为空的时候做无效查询，引入BRPOP和BLPOP

```
BRPOP  key [key ...] timeout 
BLPOP 
```

意义是进行对key的pop操作，如果key为空的话就等待{timeout}的时间，仍没有数据传入则操作结束

#### 几点注意

- BLPOP和BRPOP对多个线程的返回顺序遵循线程发出请求的顺序
- 和LPOP以及RPOP不同，BLPOP和BRPOP返回的是两个元素的array，第一个是key，第二个才是value
-  timeout到后，会返回NULL

### Automatic creation and removal of keys

在redis中，当一个key为空的时候，redis会自动将其删除（ Stream类型的数据除外），而读取一个不存在的key的时候，其返回结果就好像访问一个存在的空的key一样；同样，一个key会被创建，当这个key不存在而且我们尝试向其中插入元素；但是，当一个key已经存在的时候，插入的数据类型一定要匹配key的数据类型

### Hashes

就简单的哈希，有点像是定义了一个名称为{key}，变量名为field，值为value的结构体（？

很多命令似乎都是直接在对string的命令前面加个h

```
 hset key field value [field value ...]
 hget key field
 hmset key field value [field value ...]
 hincrby key field increment
 etc
```



## 命令大全

```bash
ping #检查redis服务是否在运行
set [key] [value] #设置键值对
get [key] #获取键对应的值
getset [key] [value] #获取旧值并设置新值
incr [key]  #值自增（对于整数）
incrby [key] [increment] #增加特定的值
exists #检查key是否存在

```

### Sets

sets是string的无序集合，支持查询某个元素是否在某个set中

```
sadd key member [member ...]
smembers key #查询key里面的所有元素
sismember key member #查询member是否在key中
sinter key [key ...] #将后面的所有key中的元素列出来
spop key [count] #随机弹出count个元素
sunionstore destination key [key ...] #把多个key放到destination里面

```

### Sorted sets

有序集合，其中的元素不能相同，每个元素由用户指定score，排序规则如下：

> - If B and A are two elements with a different score, then A > B if A.score is > B.score.
> - If B and A have exactly the same score, then A > B if the A string is lexicographically greater than the B string. B and A strings can't be equal since sorted sets only have unique elements.

```
zadd key score member [score member ...] #指定数据的score
zrange key start stop #类似lrange，升序排列
zrevrange key start stop #降序排列
zrangebyscore key min max #返回数据在min到max的值（-inf和inf表无穷）
zrank key member #取得member在key中的排名
zrevrank
```

redis 2.8还加入了按照字典序排序的功能

```
ZRANGEBYLEX, ZREVRANGEBYLEX, ZREMRANGEBYLEX ZLEXCOUNT.
```

如果想要更新元素的话，直接ZADD就行

### Bitmaps

实际上是和位运算有关的一些函数，而不是一个数据类型

```
setbit
getbit
bitop
bitcount
bitpos
```

### HyperLogLogs

一个好神奇的东西，用很少的内存来估计一组数据里面有多少不同的数据

操作上和set类似

```
pfadd key element #向key里面“添加”元素
pfcount key #看key里面有多少个不同的元素
```

神奇之处在于，pfadd并没有真正往key里面添加元素，而是用了一些神奇的数学方法保存了key的状态，从而实现**估计**key里面有多少个相异元素的功能

好神奇哦

## Redis的服务配置

挖个坑，以后再填

## 参考

[Redis data types tutorial | Redis](https://redis.io/docs/data-types/tutorial/)