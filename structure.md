**redis 的5种基本数据结构**

**string**

&#8195;字符串可以用来缓存用户信息，将用户信心json序列化成string存起来，要用的时候再将其翻序列化

&#8195;redis的字符串是动态字符串，一般长度都要比现有稍大。类似于JAVA的arraylist，采用分配冗余空间的方式来减少内存的频繁分配。当字符串长度小于1m时，扩容会加倍现有空间，当超过1m时，扩容一次只会增加1m空间。

&#8195;要注意string最大的长度为512MB

* string的使用方法

&#8195;对单个string进行的操作

```redis
127.0.0.1:6379> set name codehole
OK
127.0.0.1:6379> get name
"codehole"
127.0.0.1:6379> exists name
(integer) 1
127.0.0.1:6379> del name
(integer) 1
127.0.0.1:6379> get name
(nil)
```

&#8195;对多个string同时操作，节省网络耗时开销

```redis
127.0.0.1:6379> mset name1 boy name2 girl name3 unknown
OK
127.0.0.1:6379> mget name1 name2 name3
1) "boy"
2) "girl"
3) "unknown"
```

&#8195;**注意redis要批量删除，直接del key1 key2 key3...就行了**

&#8195;设置变量过期和set命令拓展

```redis
127.0.0.1:6379> set name codehole
OK
127.0.0.1:6379> get name
"codehole"
127.0.0.1:6379> expire name 5
(integer) 1
127.0.0.1:6379> get name
(nil)
127.0.0.1:6379>
```

&#8195;expoire就是用来设置过期时间的，数字以秒为单位

```redis
127.0.0.1:6379> setex name 5 codehole
OK
127.0.0.1:6379> get name
"codehole"
127.0.0.1:6379> get name
(nil)
```

&#8195;setex相当于set 和 expire的复合命令，setex key second value这种形式

```redis
127.0.0.1:6379> setnx name codehole
(integer) 1
127.0.0.1:6379> setnx name helloworld
(integer) 0
127.0.0.1:6379> get name
"codehole"

```

&#8195;setnx指令用于key是否存在，存在就不变，不存在就新建

&#8195;计数

```redis
127.0.0.1:6379> set age 30
OK
127.0.0.1:6379> incr age
(integer) 31
127.0.0.1:6379> incrby age 5
(integer) 36
127.0.0.1:6379> set codehole 9223372036854775807
OK
127.0.0.1:6379> incr codehole
(error) ERR increment or decrement would overflow
```

&#8195;incr是自增，incrby是指定增加，但是redis的自增范围在sigined long的最大最小值间。超出范围就会报错

&#8195;字符串由8个bit组成，如此便可以看成多个bit组合，这就是bitmap（位图）的数据结构。以后会具体讲解

**list**

&#8195;redis的list相当于JAVA里的LinkedList，它是链表，而不是数组。list的插入和删除操作非常的快，但是索引定位会非常慢。列表中的每个元素都使用双向指针顺序，串起来可以同时支持前向后向遍历。当列表弹出了最后一个元素，该数据结构被自动删除，内存被回收。

&#8195;redis的列表常用来做异步队列的使用，将需要延后处理的任务结构体序列化成字符串，塞进redis的列表，另一个线程从这个列表中轮询数据进行处理。

* redis作为队列使用（右进左出，先进先出

&#8195;队列常用于消息排队和异步逻辑处理，它会确保元素的访问顺序性

```redis
127.0.0.1:6379> rpush books python java golang
(integer) 3
127.0.0.1:6379> llen books
(integer) 3
127.0.0.1:6379> lpop books
"python"
127.0.0.1:6379> lpop books
"java"
127.0.0.1:6379> lpop books
"golang"
127.0.0.1:6379> lpop books
(nil)
```

&#8195;这里的r就是right，l就是left，lpop就是从左边出来，llen是获得list长度

* redis作为栈的使用（先进后出

&#8195;redis的list作为栈使用的业务场景并不多见

```redis
127.0.0.1:6379> rpush books python java golang
(integer) 3
127.0.0.1:6379> rpop books
"golang"
127.0.0.1:6379> rpop books
"java"
127.0.0.1:6379> rpop books
"python"
127.0.0.1:6379> rpop books
(nil)
```

&#8195;对应的，rpop就是从右边弹出。

* redis的慢操作

&#8195;lindex是获取索引对应的元素，就是lindex 1表示lindex获取第二个元素（0是第一个）。lrange 0 -1 表示索引为0和索引为-1之间的元素。索引可以为-，-1就表示倒数第一个元素。ltrim 1 -1 表示保留索引为1和倒数第一个元素之间的元素，其他全部去掉。但是有一点需要注意，**以上指令，会随着index的增加，效率会下降，慎用。**

```redis
127.0.0.1:6379> rpush books python java golang
(integer) 3
127.0.0.1:6379> lindex books 1
"java"
127.0.0.1:6379> lrange books 0 -1
1) "python"
2) "java"
3) "golang"
127.0.0.1:6379> ltrim books 1 -1
OK
127.0.0.1:6379> lrange books 0 -1
1) "java"
2) "golang"
127.0.0.1:6379> ltrim books 1 0
OK
127.0.0.1:6379> llen books
(integer) 0
```

&#8195;ltrim ... 1 0的保留的长度为-1，相当于清空list

* 快速列表

&#8195;list的底层存储是一个叫快速链表的结构。

&#8195;list元素较少的情况下，会使用一块连续的内存存储，这个结构是ziplist，压缩列表。它将所有元素彼此挨在一起存储，分配的是一块连续的内存。当数据较多时才会改成quicklist。普通的链表要附加指针空间太大，会浪费空间和加重内存的碎片化。而redis的quicklist其实就是用指针连接多个ziplist，满足了快速删插，也不会出现太大的空间冗余。

**hash**

* hash的基本操作

&#8195;hash是无序字典，内部存储了很多建值对，实现结构是数组+链表。redis的值只能是字符串。redis的rehash采用渐进式rehash，rehash时，保留新旧两个hash结构，查询会同时查询两个hash结构，然后在后续的定时任务及hash操作指令中，循序渐进的将旧的hash内容一点点迁移到新的hash中去。搬运完后，就会使用新的hash结构取而代之。

&#8195;当hash移除了最后一个元素之后，该数据结构会被删除，内存会被回收

&#8195;hash结构也可以用来存储用户信息，与字符串需要一次性全部序列化整个对象，不同，hash可以对用户结构中的每个字段单独存储。这样可以部分获取，而字符串只能一次性全部获取，这样会浪费网络流量。但是hash结构存储消耗要高于单个字符串，要用哪个还是需要再权衡。

```redis
127.0.0.1:6379> hset books java "think in java"
(integer) 1
127.0.0.1:6379> hset books python "python cookbook"
(integer) 1
127.0.0.1:6379> hset books golang "concurrency in go"
(integer) 1
127.0.0.1:6379> hgetall books
1) "java"
2) "think in java"
3) "python"
4) "python cookbook"
5) "golang"
6) "concurrency in go"
127.0.0.1:6379> hlen books
(integer) 3
127.0.0.1:6379> hget books java
"think in java"
127.0.0.1:6379> hset books golang "learning go programming"
(integer) 0
127.0.0.1:6379> hget books golang
"learning go programming"
127.0.0.1:6379> hmset books java "effective java" python "lerning python"
OK
127.0.0.1:6379> hgetall books
1) "java"
2) "effective java"
3) "python"
4) "lerning python"
5) "golang"
6) "learning go programming"
127.0.0.1:6379> del books
(integer) 1
127.0.0.1:6379> hgetall books
(empty array)
```

&#8195;这里用hget key来创建hash，用hget key field 获取成员，用hgetall key 获取整个hash，结果一个field一个value排列。用hmset来批量操作。hlen来获得field的个数。del来删除这个hash。


* 计数

&#8195;和字符串一样，hash结构中的单个key也可以进行计数，它对应的指令是hincrby，和incr的使用方法基本一致。

```redis
127.0.0.1:6379> hset user-laoqian age 29
(integer) 1
127.0.0.1:6379> hincrby user-laoqian age 1
(integer) 30
```

**set集合**

&#8195;set内部建值对是无序的，唯一的。是一个特殊的字典。字典中所有的value都是一个值NULL。当集合中最后一个元素被移除后，数据结构会被删除，内存被回收。

&#8195;set可以用于存储中奖用户ID，因为set有去重功能，可以保证同一个用户不会中奖两次。

```redis
127.0.0.1:6379> sadd books python
(integer) 1
127.0.0.1:6379> sadd books python
(integer) 0
127.0.0.1:6379> sadd books java golang
(integer) 2
127.0.0.1:6379> smembers books
1) "java"
2) "python"
3) "golang"
127.0.0.1:6379> sismember books java
(integer) 1
127.0.0.1:6379> sismember books rust
(integer) 0
127.0.0.1:6379> scard books
(integer) 3
127.0.0.1:6379> spop books
"java"
```

&#8195;set没有value，只有key，这里和hash很大的不一样。可以看见重复的key插入返回0，是失败的。sadd新建set。smembers获取set，也可以获取部分。注意查询结果顺序不一样，是乱序的。sismember查key是否存在。scard获取元素个数。spop弹出一个元素。同时也可以del删除。

&#8195;要注意如果有空格，要用""括起来

**zset（有序列表**

* zset基础操作

&#8195;同理，zset最后一个元素被移除，该数据结构会被删除，内存被回收。zset一方面，它是一个set，保证value的的唯一性，另一方面，它可以给每一个value赋予一个score，代表这个value的排序权重。内部实现是叫做跳跃列表的数据结构。

&#8195;zset常用来存储学生的成绩，value代表学生ID，score是考试成绩，对成绩排序就可以得到名次。

```redis
127.0.0.1:6379> zadd books 8.9 "java concurrency"
(integer) 1
127.0.0.1:6379> zadd books 8.6 "java cookbook"
(integer) 1
127.0.0.1:6379> zrange books 0 -1
1) "java cookbook"
2) "java concurrency"
3) "think in java"
127.0.0.1:6379> zrevrange books 0 -1
1) "think in java"
2) "java concurrency"
3) "java cookbook"
127.0.0.1:6379> zcard books
(integer) 3
127.0.0.1:6379> zscore books "java concurrency"
"8.9000000000000004"
127.0.0.1:6379> zrank books "java concurrency"
(integer) 1
127.0.0.1:6379> zrangebyscore books 0 8.91
1) "java cookbook"
2) "java concurrency"
127.0.0.1:6379> zrangebyscore books -inf 8.91 withscores
1) "java cookbook"
2) "8.5999999999999996"
3) "java concurrency"
4) "8.9000000000000004"
127.0.0.1:6379> zrem books "java concurrency"
(integer) 1
127.0.0.1:6379> zrange books 0 -1
1) "java cookbook"
2) "think in java"
```

&#8195;zadd可以用来新建一个有序列表，zrange用来排序列出，参数区间表示排名范围。zrevrange是按score逆序列出。参数区间是排名范围。zcard获得key的个数。zscore获得指定value的score。内部的score使用double类型存储。zrank可以排名。**（怎么个排名不清楚**。zrangebyscore 根据分值区间遍历。inf表示无穷大，-inf表示负无穷大。 withscores可以带着分数一起输出。zrem删除value。

* 跳跃列表

&#8195;跳跃列表就是普通链表，下面列出层级，然后层级也用链表连起来。当插入元素时，就通过层级寻找。


**容器型数据结构的通用规则**

* 容器如果不存在，就创建一个

* 容器如果没有元素，就删除，释放内存

**过期时间**

* 过期时间是整个对象的过期。hash的过期时间是整个hash的过期，而不是hash中某一个key的过期时间。

* 如果设置了过期时间，又调用了set去修改，那么过期时间会消失。


