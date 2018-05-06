title: 通过 Redlock 实现分布式锁
date: 2018-05-06 01:42:00
tags:
- Distributed lock
- Redis
- Redlock
categories:
- Develop
---

处理跑在多台机器上的程序间的资源竞争，能想到的最简单的方法就是加分布式锁。前年在 A 实习的时候用到了 [Tair 做的分布式锁](https://yq.aliyun.com/articles/58928)，自然就联想到同是 KV 数据库 Redis 大约也可以做分布式锁来使用。

Redis 在官方分布式锁的 topic 中提出了一种分布式锁算法 -- Redlock，相对其他的 Redis 分布式锁实现，会更加可靠一些（起码在单机上），同时又支持多个实例。

# Redlock 算法概要

## 设计目标

Redlock 在设计之初提出了 3 个特性来保证分布式锁可以高效、正常的实现：

1. 安全性：在统一时间内，只有一个客户端可以锁。
2. 可用性 A：死锁避免，锁可以保证最终被释放。
3. 可用性 B：容错性，只要大部分 Redis 节点启动，客户端可以获取和释放锁。

## 简单单机实现和 Failover 多机实现的问题

### 基于 SETNX 的简单单机实现

网上有很多 Redis 锁的简单单机实现，大部分为通过 `SETNX` 设置一个自定义过期时间的 Key Value，释放锁是删除 Key Value，同时由于过期时间锁的最终释放是可以保证的。

由于未验证 Key 对应 Value 的值，可能在以下场景中存在问题：

1. 客户端 A 从服务器上获取锁。
2. 客户端 A 占有锁的时间超时，但仍在执行。
3. 客户端 B 获取锁。
4. 客户端 A 释放锁，由于没有锁的拥有者信息，可直接删除 Key 对应的 Value。
5. 客户端 B 仍认为自己拥有锁，此刻客户端 C 可获取锁。

### Failover 多机实现的问题

如果在正确单机实现的基础上，使用增加 Slave 的形式来保证可用性，由于 Master 与 Slave 节点之间的复制是异步的，故在一下场景中会存在问题：

1. 客户端 A 从 Master 上获取锁。
2. 在锁未被复制到某 Slave 节点的时候，Master 节点 Down 掉了。
3. 某 Slave 节点成为新的 Master。
4. 客户端 B 可从新 Master 上获取锁。

## Redlock 算法

### 单机分布式锁的正确实现

在实现多机的分布式锁算法之前，需要一个可用的单机分布式锁。

实现还是主要基于 `SET` 的 `NX` 和超时要素，针对不验证就 `DEL` 的不安全问题，可以采用类似签名的方法，即在获取锁时对 Key 对应的 Value 设置一个随机值，释放锁的时候再进行验证。

签名的随机值从简单来说可以使用客户端 ID 加上时间戳，对可靠性要求高的话可以使用经过 `RC4` 算法生成的[随机数流](https://en.wikipedia.org/wiki/RC4#Pseudo-random_generation_algorithm_%28PRGA%29)，种子使用 `/dev/urandom` 产生的随机值即可。

具体的释放锁脚本通过 Lua 编写，以脚本形式执行，由于单个 Redis 实例采用单个 Lua 解释器运行脚本，所以可以认为该脚本的执行过程是原子的：

```lua
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
```

由于需要保证锁的最终释放，还是需要对锁设置时间，`SET` 中超时的时间即为锁合法的时间，超过该时间锁就可以被其他的客户端获取。

### Redlock 算法

Redlock 使用的 Redis 节点皆为 Master 节点，以 5 个节点为例子，具体的实现如下：

1. 客户端获取当前的时间戳。
2. 对 N 个 Redis 实例进行获取锁的操作，具体的操作同单机分布式锁。对 Redis 实例的操作时间需要远小于分布式锁的超时时间，这样可以保证在少数 Redis 节点 Down 掉的时候仍可快速对下一个节点进行操作。
3. 客户端会记录所有实例返回加锁成功的时间，只有从多半的实例（在这里例子中 >= 3）获取到了锁，且操作的时间远小于分布式锁的超时时间，锁才被人为是正确获取。
4. 如果锁被成功获取了，当前分布式锁的合法时间为初始设定的合法时间减去上锁所花的时间。
5. 若分布式锁获取失败，会强制对所有实例进行锁释放的操作，即使这个实例上不存在相应的键值。

### 一些细节

#### 非同步的时钟

由于 Redlock 算法中使用的各个实例没有同步的时钟，而且每个机器时钟可能也不准，简单的做法是对步骤 3 中的时间再减去一小段时间，从而缓解实际中各个机器间的时钟偏移（Clock Drift）。

#### 失败重试

若获取锁失败，会随机等一段时间再获取锁，以免竞争导致每个客户端都无法获取锁。理想情况下，应该使用多路复用同时对 N 个实例进行请求。

为了减少客户端等待时间，获取锁失败后会立即释放，相对等待超时效率更高。

若因网络原因无法访问 Redis 实例，应该增加等待时间。

#### 性能、故障恢复和 fsync

假设 Redis 没有持久性，当一个客户端获得了 5 个实例中的 3 个锁，若 3 个锁所在的实例 Down 掉了，实例再次启动时，其他的客户端也可以再次获得锁。

这个问题会因为开启了 Redis 的持久化而改观，对于 AOF 持久化（区别与 RDB 的二进制持久化，是文本持久化）。默认采用的是每秒钟通过 `fsync` 落盘，这意味着会丢失一秒内的数据，如果需要更有安全保证的持久化，可以设置 `fsync=always`，但对应的会损失一部分性能。

更好的解决办法是在实例 Down 掉后延迟一个略长于锁合法时间的时间，这样就可以保证在实例启动起来时锁一定是过期的，从而无须以损失性能为代价而使用 `fsync=always` 的持久化。

#### 锁续约

Topic 中还提出了一个锁续约的概念，即客户端可初始时使用较小的锁有效时间，若时间超时仍操作仍未完成则通过 Lua 脚本来续期。

续约操作同获取锁的操作，如果能从多数的 Redis 实例中续约，则认为锁续约成功。

为了保证锁最终可以被释放，续约操作存在着一定的次数限制。

### Redlock 设计目标论证

#### 安全性

假设客户端可以获取大部分实例上的锁，而所有实例均有一个相同的锁合法时间，由于初始设置的时间不一定相同，所以各个实例上的键不一定同时过期。假设 `T1` 为请求第一个实例的时间，`T2` 为从最后一个服务器得到回复的时间。在客户端认为获得到分布式锁后，第一个实例上的键会在 `MIN_VALIDITY = TTL - (T2 - T1) - CLOCK_DRIFT` 后过期，之后其他的键也会过期，所以确定在 `MIN_VALIDITY` 内，所有键都还未过期。

在过半实例都被设置的情况下，若一个客户端请求获取锁，因为无法在过半的实例上设置键，所以会失败。如果一个客户端占有锁的时间接近或大于 `MIN_VALIDITY`，部分实例上的锁才会失效。`MIN_VALIDITY` 内是不可能存在过半实例可以被设置键的情况，故在 `MIN_VALIDITY` 内分布式锁是安全的。

#### 可用性

系统的可用性由以下几点保证：

1. 因为过期机制，锁一定会最终被释放。
2. 为了更好的效率，在使用完锁之后会将锁删除。
3. 如果客户端失败后要重新获取锁，等待时间须大于所有锁的时间，从而避免客户端抢占锁，最后导致锁无法获取的情况。

针对第三点，如果在获取锁失败后，删除锁的过程中由于网络超时，需要增加额外的长为 TTL（即键的合法时间）的惩罚，这样可以避免无法删除锁而导致的重试时间的浪费。

# Redlock 相关的讨论

针对 Redlock 的安全性问题，[Martin Kleppmann](http://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) 与作者 [antirez](http://antirez.com/news/101) 存在着一些争论。

## Redlock 存在的问题

Martin Kleppmann 提出使用分布式锁有两种目的：效率和正确性。

### 效率

效率相关的应用主要在使得同一个运算不会被同时计算两次，如果在这种情况下使用多个实例的分布式锁，不如使用单机分布式锁实现。由于误判仅会增加计算量而并不影响正确性，所以可以使用 Master Slave 的方式来增加可用性。

### 正确性

正确性相关的质疑在两个方面：

1. 由于无法保证锁在操作结束前一定不会结束，可能无法意识到自己的锁失效了，会导致同时有多个客户端持有锁并进行操作。
2. 基于时间的假设是不可靠的。

针对第二点，Martin 举了两个例子，以下均以 5 个节点 A ~ E 和 2 个客户端为例。

1. Redis 实例的时间无法保证不被修改，时间的修改可能导致锁的过期和异常：

    1. 客户端 1 从节点 A、B、C 上获取锁，节点 D、E 由于网络原因不可达。
    2. 节点 C 的时间跳变，锁过期了。
    3. 客户端 2 从节点 C、D、E 上获取了锁，由于网络原因，节点 A、B 不可达。
    4. 客户端 1、2 均认为它们拥有了锁。

2. 即使时间不会跳变，进程停等也可能影响正确性：

    1. 客户端 1 从节点 A、B、C、D、E 上获取了锁。
    2. 当请求快抵达客户端 1 时，客户端 GC 进行了 stop-the-world 操作。
    3. 所有实例上的锁都过期了。
    4. 客户端 2 从节点 A、B、C、D、E 上获取了锁。
    5. 客户端 1 GC 结束了，同时获取了之前的请求信息。
    6. 客户端 1、2 均认为它们拥有了锁。

Martin 提出使用一种小 trick，使用类似的递增 ID 作为 fencing token 来保证每次锁内操作的正确性。在客户端每次请求中会检查 fencing token，如果依赖的是旧的 token，那么此操作会被拒绝并提示。

Martin 认为 Redlock 是一个 [asynchronous model with unreliable failure detectors](http://courses.csail.mit.edu/6.852/08/papers/CT96-JACM.pdf)，依赖了不可靠的元素来保证其一致性，容错存在一个限度，超过限度则无法保证其正确性。GC 的 stop-the-world、网络的阻塞、换页等都会影响 Redlock 的效率或者正确性。 

Martin 的结论是：如果仅是效率需求，使用单节点算法；如果需求正确性，通过 zookeeper 来实现锁，或者使用 fencing token 来保证安全性。

## 解释和改进

antirez 解释了目的是为了让人们找到单节点或者主备锁的替代品，从而在复杂度低以及高效的前提下使用锁。

针对正确性的第一点质疑，即锁内可能超时，antirez 认为分布式锁并不能提供强一致性的保证，只在没有别的办法控制共享资源时才会使用。

针对正确性的第二点质疑，对于第一个例子，提出 NTP 和手动修改时间导致的时间过大跃迁都可以被人工避免，其次也提出可以使用单调时间 API 来进行改进（不过好像至今没有更新）；对于第二个例子，实际出现次数非常少，另可以通过在获取到锁之后检查获取锁之前后的本地时间来确定锁是否过期。

对于 fencing token，若操作之间存在着线性关系，则可更改为递增 id；如果没有线性关系，则可以每次操作前检查 Key 对应的随机 Value 来避免过期操作。

# 总结

对于增强效率的应用场景，使用单机分布式锁加上 Master Slave 基本够用；对于正确性相关的应用场景，还是算需要具体分析吧，对于解决资源之间的竞态，改进后的 Redlock 应该就够用；如果设计到分布式事务、操作间需要协调一类的场景，尽管 Redlock 可以通过增加 fencing token 等改进增强其可靠性，不过就像 antirez 说的，Redlock 并不是设计用来保持强一致性的，对于强一致性需求可以选择更合适的框架，例如 Zookeeper。

--------------

参考资料：
1. https://yq.aliyun.com/articles/58928
2. https://redis.io/topics/distlock
3. http://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html
4. http://antirez.com/news/101