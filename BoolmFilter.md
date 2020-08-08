布隆过滤器多用于对大量数据 exit 查找是否存在 消耗少，但是有误差

布隆过滤器可以理解为一个不怎么精确的 set 有可能会误判。

当它说这个值存在时，可能不存在。说一个值不存在时，一定不存在。

布隆过滤器通过运行插件来使用：sudo docker run -p6379:6379 redislabs/rebloom

* 布隆过滤器有两个基本指令。 bf.add 和 bf.exists 前者添加元素，后者判断元素是否存在。用法和set的sadd和sismember差不多。注意bf.add一次只能添加一个元素，多个需要用bf.madd和bf.mexists

```redis
127.0.0.1:6379> bf.add codehole uuser1
(integer) 1
127.0.0.1:6379> del codehole
(integer) 1
127.0.0.1:6379> bf.add codehole user1
(integer) 1
127.0.0.1:6379> bf.add codehole user2
(integer) 1
127.0.0.1:6379> bf.exists codehole user1
(integer) 1
127.0.0.1:6379> bf.exists codehole user4
(integer) 0
```

* 降低误判率： 改变bf.reserve 的三个参数，key error_rate initial_size

error_rate越低，占用空间越高。initial_size是预计放入元素的大小，当超过这个值时，误判率会变高。

initial_size 设置过大会浪费空间，设置过小会增加误判率。一定要估计好量，并且设置一定冗余

error_rate 在不需要非常精确的场合，设置大一点也没有关系

* 布隆过滤器原理: 实际上就是一个大型位数组，和几个无偏hash函数。加入key时，三个函数算出索引，然后在对应的位数组上重置为1。判断key是否存在，就是通过hash函数算，判断几个位置是否是1。所以存在的一定存在。但是判断不存在的时候，可能原来是0的位置，被别的key重置成了1，所以可能会误判。当位数组比较稀疏，误判概率越小，当位数组越拥挤，误判概率越大。

注意不要让实际数据远大于初始化数据。当大于的时候，要重新分配一个size更大的数组，然后再将原来的数据add到里面去。因为刚开始超过，误判率不会大幅提升，所以会有时间重置。

* 布隆过滤器的占用空间可以通过网站计算 https://krisives.github.io/bloom-calculator

* 布隆过滤器还会用在邮箱，数据库，爬虫等


