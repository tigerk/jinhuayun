Redis官方文档中是采用RedLock来实现的，是redis官方支持的分布式锁算法。

### 什么是 RedLock
Redis 官方站这篇文章提出了一种权威的基于 Redis 实现分布式锁的方式名叫 Redlock。

redLock有三个特性：

* **安全特性** ：互斥访问，即永远只有一个 client 能拿到锁。
* **避免死锁** ：最终 client 都可能拿到锁，不会出现死锁的情况，即使原本锁住某资源的 client crash 了或者出现了网络分区。
* **容错性** ：只要大部分 Redis 节点存活就可以正常提供服务。

### RedLock算法

有5个master节点，分布在不同的机房保证可用性，为了获得锁，client会进行如下操作
* 获取当前时间戳，单位是毫秒；
* 尝试顺序地在 5 个实例上申请锁，当然需要使用相同的 key 和 random value，这里一个 client 需要合理设置与 master 节点沟通的 timeout 大小，避免长时间和一个 fail 了的节点浪费时间。
* 当 client 在大于等于 3 个 master 上成功申请到锁的时候，且它会计算申请锁消耗了多少时间，这部分消耗的时间采用获得锁的当下时间减去第一步获得的时间戳得到，如果锁的持续时长（lock validity time）比流逝的时间多的话，那么锁就真正获取到了。
* 如果锁申请到了，那么锁真正的 lock validity time 应该是 origin（lock validity time） - 申请锁期间流逝的时间
* 如果 client 申请锁失败了，那么它就会在 master 节点上执行释放锁的操作，重置状态

- 失败重试
一个client申请失败，他需要随机时间后重试避免多个client同时申请锁，防止多个客户端在同事争夺资源时发生脑裂，最好是一个client同时向5个master申请锁。如果 client 申请锁失败了它需要尽快在 master 上执行 unlock 操作。便于其他client获取锁。

> 未能获取n/2 + 1 个锁的client需要尽快释放锁，这样其他客户端不需要等待到期时间再次获取锁，如果网络原因导致无法释放，那就不得不付出代价了。

> **脑裂(split brain)：**
> 三个Client同时尝试获得锁，分别获得了2, 2, 1个实例中的锁，三个锁请求全部失败。
> 一个client在全部Redis实例中完成的申请时间越短，发生脑裂的时间窗口越小。
> 所以比较理想的做法：同时向N个Redis实例发出异步的SET请求。

- 释放锁

释放锁比较简单，向所有的Redis实例发送释放锁命令即可，不用关心之前有没有从Redis实例成功获取到锁。

- 性能，崩溃恢复和fsync

使用redis锁需要在获取和释放锁时保证低延时，所以要与多个redis通信时要采用并发方式来解决。在考虑崩溃恢复时要注意持久性问题。



**举个栗子** ：

在节点没有持久化时，client从5个master获取了3个锁，然后其中一个重启了，这时又有一个client可以获取三个锁。



以上情况违反了互斥性。如果有AOF持久化会改善，因为redis过期是在语义上的，但是断电情况下，AOF没有刷到硬盘就丢失了怎么办？fsync=always 会有性能问题。

所以在一个节点重启之后，让节点在max TTL期间是不可用的，这样就不会影响已经申请到的锁，等到宕机前的锁都过期了，我们在把节点加进来工作但是当大部分redis都挂了，就会出现没有资源可用的情况。



### But 这里有个前提

RedLock算法基于这样一个假设：虽然多个进程之间没有时钟同步，但每个进程都以相同的时钟频率前进，时间差相对于失效时间来说几乎可以忽略不计。每个计算机都有一个本地时钟，我们可以容忍多个计算机之间有较小的时钟漂移。