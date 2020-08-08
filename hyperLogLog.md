HyperLogLog 是redis的一个高级数据结构，可以应用于记去重的大量用户登陆记录。如果是浏览记录，因为不用存储uid，直接用incrby就行。

HyperLogLog 存在误差，误差在0.81%内

* 两个指令，pfadd(增加计数) pfcount(获取计数)

```redis
127.0.0.1:6379> pfadd codehole user1
(integer) 1
127.0.0.1:6379> pfcount codehole
(integer) 1
127.0.0.1:6379> pfadd codehole user2
(integer) 1
127.0.0.1:6379> pfcount codehole
(integer) 2
127.0.0.1:6379> pfadd codehole user3
(integer) 1
127.0.0.1:6379> pfcount codehole
(integer) 3
127.0.0.1:6379> pfadd codehole user4
(integer) 1
127.0.0.1:6379> pfcount codehole
(integer) 4
```

* pfmerge 合并两个 pf 成为同一个 pf

* HyperLogLog 需要消耗12kb的空间，不适合用来存储单个用户的数据，多用户比起来比 set 的消耗少

&#8195;redis 的 HyperLogLog 存储经过了优化，计算数据较小的时候，会用比较稀疏矩阵存储，当数据超过了阀值，才会占用12kb。

* HyperLogLog 的原理大概是存储未知数，通过算法来计算数量。该数据结构用了16384个桶进行未知数计算。每个桐的 maxbit 是6个 bit 。最大表示 maxbit=63 。算出来就是12kb

HLL用的是伯努利算法
