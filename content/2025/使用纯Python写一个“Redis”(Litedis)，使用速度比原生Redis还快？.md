---
title: "使用纯Python写一个“Redis”(Litedis)，使用速度比原生Redis还快？"
date: 2025-01-14 17:18:54

categories:
  - 数据库
  - Python
tags:
  - Redis
  - Python
  - 键值存储
  - NoSQL

summary: "介绍了 Litedis 项目的功能特性，并将 Litedis 与 Redis 在使用速度方面进行对比，得出 Litedis 的性能情况。"
---

最近使用 Python 模仿 Redis 写了一个类似的 NoSQL，起名叫 `Litedis`。本意是想实现一个带有 Redis 简单功能，同时是轻量的、本地的，可以开箱即用的 NoSQL 数据库。

Python 是执行速度垫底的编程语言，C 可以认为是最快的语言。Redis 是 C 语言编写的项目，使用 Python 编写的 Litedis 项目在执行速度上当然不可能快过它。题目中说的是“使用速度”，指的是平常的使用速度。经过测试，Litedis 的平常使用速度比 Redis 要快很多。那么快多少呢？

这里先卖个关子，先来介绍一下 Litedis 项目，然后再介绍两者的使用速度对比。想直接看对比的读者可以直接跳到对比的部分阅读。

## Litedis

如前所说，Litedis 是一个轻量的、本地的、开箱即用的 NoSQL，实现了简单的 Redis 功能。使用 Litedis 的时候，安装后导入就可以直接使用了，不用像 Redis 那样需要单独开一个服务器进程。

有些时候，我们是可以不必用到 Redis 服务的。比如调试代码的时候，只是想测试一下代码能否正常运行；又比如有时我们只是想验证一些想法，如果想法不可行，那么该项目就可以丢弃了；又比如有时我们只是单纯想要一个可以本地使用的、带有持久化功能的 NoSQL 数据库等等。

上面提到的这些情况就可以使用 Litedis，等到真正需要 Redis 的时候，再切换过去就行了。切换成本是很低的，因为 Litedis 面向用户的接口是参照有名的 Redis 客户端 `redis-py` 项目编写的，除了有些参数名不同，函数名是一样的。

下面介绍一下 Litedis 的特性和一些使用示例。

### Litedis 特性

- 实现了基础数据结构及相关操作：  
  - STING  
  - LIST  
  - HASH  
  - SET  
  - ZSET  
- 支持设置过期时间  
- 支持 AOF 持久化

### Litedis 使用示例
#### 持久化和数据库设置  
  
- Litedis 是默认开启持久化的，可以通过参数进行相关设置。  
- Litedis 可以将数据存储到不同的数据库下，可以通过 dbname 参数设置  
  
```python  
from litedis import Litedis  
  
# 关闭持久化  
litedis = Litedis(persistence_on=False)  
  
# 设置持久化路径  
litedis = Litedis(data_path="path")  
  
# 设置数据库名称  
litedis = Litedis(dbname="litedis")  
```  
  
#### STRING 的使用  

```python  
import time  
  
from litedis import Litedis  
  
litedis = Litedis()  
  
# set and get  
litedis.set("db", "litedis")  
assert litedis.get("db") == "litedis"  
  
# delete  
litedis.delete("db")  
assert litedis.get("db") is None  
  
# expiration  
litedis.set("db", "litedis", px=100)  # 100毫秒后过期  
assert litedis.get("db") == "litedis"  
time.sleep(0.11)  
assert litedis.get("db") is None  
```  

#### LIST 的使用    
  
```python  
from litedis import Litedis  
  
litedis = Litedis()  
  
# lpush  
litedis.lpush("list", "a", "b", "c")  
assert litedis.lrange("list", 0, -1) == ["c", "b", "a"]  
litedis.delete("list")  

# rpush  
litedis.rpush("list", "a", "b", "c")  
assert litedis.lrange("list", 0, -1) == ["a", "b", "c"]  
litedis.delete("list")  
  
# lpop  
litedis.lpush("list", "a", "b")  
assert litedis.lpop("list") == "b"  
assert litedis.lpop("list") == "a"  
assert litedis.lrange("list", 0, -1) == []  
assert not litedis.exists("list")  # 当所有元素被弹出后，相应的 List键 会自动删除  
```  
  
#### Hash 的使用  
  
```python  
from litedis import Litedis  
  
litedis = Litedis()  
litedis.delete("hash")  
  
# hset  
litedis.hset("hash", {"key1":"value1", "key2":"value2"})  
assert litedis.hget("hash", "key1") == "value1"  
  
# hkeys and hvals  
assert litedis.hkeys("hash") == ["key1", "key2"]  
assert litedis.hvals("hash") == ["value1", "value2"]  
```  
  
#### SET 的使用  
  
```python  
from litedis import Litedis  
  
litedis = Litedis()  
litedis.delete("set", "set1", "set2")  
  
# sadd  
litedis.sadd("set", "a")  
litedis.sadd("set", "b", "c")  
members = litedis.smembers("set")  
assert set(members) == {"a", "b", "c"}  
  
litedis.sadd("set1", "a", "b", "c")  
litedis.sadd("set2", "b", "c", "d")  
  
# inter  
result = litedis.sinter("set1", "set2")  
assert set(result) == {"b", "c"}  
# union  
result = litedis.sunion("set1", "set2")  
assert set(result) == {"a", "b", "c", "d"}  
# diff  
result = litedis.sdiff("set1", "set2")  
assert set(result) == {"a"}  
```  
  
#### ZSET 的使用  
  
```python  
from litedis import Litedis  
  
litedis = Litedis()  
litedis.delete("zset")  
  
# zadd  
litedis.zadd("zset", {"a": 1, "b": 2, "c": 3})  
assert litedis.zscore("zset", "a") == 1  
  
# zrange  
assert litedis.zrange("zset", 0, -1) == ["a", "b", "c"]  
  
# zcard  
assert litedis.zcard("zset") == 3  
  
# zscore  
assert litedis.zscore("zset", "a") == 1  
```

## Litedis 和 Redis 使用速度对比

现在来讲讲 Litedis 和 Redis 的使用速度对比。对比方法是每次使用循环写入 set 命令 10000次，共进行 5次操作。

### 测试环境

- 硬件：Mac mini M2 Pro
- 系统：macOS 15.2
- 开发工具：PyCharm 2024.3
- Python版本：Python3.10

### 使用速度
#### Litedis 的使用速度

测试代码：
```python
from timeit import timeit  
  
from litedis import Litedis  
  
lds = Litedis()  
  
def string_operations():  
    for i in range(10000):  
        lds.set(f'user:name{i}', '张三')  
  
result = timeit(string_operations, number=5)  
print(result)
```

输出：
```sh
0.2423830000043381
```

即 Litedis 写入5万次 set 命令用时 242.38 毫秒

#### Redis 的使用速度

因为笔者一般是使用 Docker 里的 Redis，所以下面测试连接的是部署在 Docker 中的 Redis。Docker 环境是使用 OrbStack(1.9.4) 的 Docker 环境。

Redis 的部署命令是：

```sh
docker run --name redis -d -p 6379:6379 redis
```

Redis 的版本是 7.4.2，redis-py 的版本是 5.2.1

测试代码：

```python
from timeit import timeit  
  
import redis  
  
rds = redis.Redis(  
    host='localhost',  
    port=6379,  
    db=0,  
    decode_responses=True  
)  
  
def string_operations():  
    for i in range(10000):  
        rds.set(f'user:name{i}', '张三')  
  
result = timeit(string_operations, number=5)  
print(result)
```

输出：
```
5.552594125008909
```

即 Redis 写入5万次 set 命令的用时是 5552.59 毫秒！

### 使用速度对比

从前面 Litedis 和 Redis 写入5万次 set 命令的用时可以看出，Litedis 的使用速度比 Redis 快了 `5552.59 / 242.38 ≈ 22.9` 倍。

### 直接运行于系统的 Redis 的速度

上面的 Redis 是运行在 Docker 中的，Docker 用起来方便，但也性能差一点。如果将 Redis 直接部署于系统中，速度会如何呢？

使用 brew 命令安装和启动 Redis

```sh
brew install redis
brew services start redis
```

然后再跑一下 Redis 的测试代码，输出：

```
1.6179862919962034
```

用时 1618 毫秒，可见比 Docker 中的 Redis 确实快了不少，快了3倍多。然而比起 Litedis，还是慢了，慢了 `1618 / 242.38 ≈ 6.7` 倍

## 总结

Litedis 虽然是用纯 Python 编写的，但实际使用性能并不低，性能方面无需有顾虑。当然了，作为一款轻量级的本地 NoSQL 数据库，其他方面能否经得起时间的考验，有待验证。

## Litedis 项目地址
[Litedis](https://github.com/linsuiyuan/litedis)