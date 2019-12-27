### Redis基本使用
### 1. Redis特点
1. Redis本质是一个key-value类型的内存数据库，很像Memcached（memcache的数据仅保存在内存中，服务器重启后，数据将丢失），整个数据库都加载在内存中进行操作，定期通过异步操作把数据库数据flush到硬盘上进行保存。正因为它是纯内存操作，Redis的性能非常出色，美妙可以处理超过十万次读写操作，是已知性能最快的key-value DB
2. Redis支持多种数据结构：String、list、set、sorted set和hash。
>下列这些数据类型都可作为值类型：
二进制安全的字符串   
Lists: 按插入顺序排序的字符串元素的集合。他们基本上就是链表（linked lists）。    
Sets: 不重复且无序的字符串元素的集合。   
Sorted sets:类似Sets,但是每个字符串元素都关联到一个叫score浮动数值（floating number value）。里面的元素总是通过score进行着排序，所以不同的是，它是可以检索的一系列元素。（例如你可能会问：给我前面10个或者后面10个元素）。Hashes,由field和关联的value组成的map。field和value都是字符串的。这和Ruby、Python的hashes很像。    
Bit arrays(或者说 simply bitmaps): 通过特殊的命令，你可以将 String 值当作一系列 bits 处理：可以设置和清除单独的 bits，数出所有设为 1 的 bits 的数量，找到最前的被设为 1 或 0 的 bit，等等。    
HyperLogLogs: 这是被用于估计一个 set 中元素数量的概率性的数据结构。    



Redis key值是二进制安全的，这意味着可以用任何二进制序列作为key值，从形如”foo”的简单字符串到一个JPEG文件的内容都可以。空字符串也是有效key值。    

#### key(键)

Redis键命令语法：command keyname
>命令集：
>1. del key：key存在时删除key
>2. dump key：序列化给定key，并返回被序列化的值
>3. exists key： 检查给定key是否存在
>4. expire key seconds：给key设置过期时间
>5. pexpire key milliseconds：设置key的过期时间，以毫秒为单位。
>6. expireat key timestamp：该命令作用和expire类似，都用于给key设置过期时间戳，不同在于expireat命令接收的时间参数是UNIX时间戳
>7. pexpireat key milliseconds：设置key的过期时间戳，以毫秒为单位
>8. persist key：移除key的过期时间，key将持久保持
>9. pttl key：以毫秒为单位返回key的剩余的过期时间
>10. ttl key（time to live）：以秒为单位，返回key的剩余生存时间
>11. rename key newkey：修改key的名称
>12. renamenx key newkey：仅当newkey不存在时，将key改名为newkey
>13. type key：返回key所存储的值的类型


下面重点介绍五种数据类型
##### 1.1. Stirng
这是Redis最简单的数据类型，如果只用这种数据类型，Redis就跟Memcached服务器没什么区别。
Redis通常使用set command和get command来设置和获取字符串值
```
C:\Users\houyl>redis-cli.exe
127.0.0.1:6379> set yolyn male
OK
127.0.0.1:6379> get yolyn
"male"
127.0.0.1:6379>
```
值可以是任意种类的字符串（包括二进制数据），甚至可以在一个键下保存一副图片，只要大小不要超过512MB。另外当key存在是set操作会失败。



String的神奇操作：

1. incr 命令将字符串值解析成整型，将其加一，最后将结果保存为新的字符串值。
类似的命令有incrby, decr 和 decrby。实际上他们在内部就是同一个命令，只是看上去有点儿不同。
```
127.0.0.1:6379>  set age 23
OK
127.0.0.1:6379> incr age
(integer) 24
127.0.0.1:6379> incrby age 100
(integer) 124
127.0.0.1:6379> decr age
(integer) 123
127.0.0.1:6379> decrby age 100
(integer) 23
127.0.0.1:6379>
```

incr是原子操作意味着什么呢？就是说即使多个客户端对同一个key发出incr命令，也决不会导致竞争的情况。例如如下情况永远不可能发生：『客户端1和客户端2同时读出“10”，他们俩都对其加到11，然后将新值设置为11』。最终的值一定是12，read-increment-set操作完成时，其他客户端不会在同一时间执行任何命令。

对字符串，另一个的令人感兴趣的操作是GETSET命令，行如其名：他为key设置新值并且返回原值。这种需求暂时没遇见过。

2. 可以一次存储或获取多个key对应的值，使用MSET和MGET命令
```
127.0.0.1:6379> mset yolyn female age 23 job java
OK
127.0.0.1:6379> mget yolyn age job
1) "female"
2) "23"
3) "java"
127.0.0.1:6379>
```

MGET 命令返回由值组成的数组


##### 1.2 Lists
Redis lists基于Linked Lists实现。这意味着即使在一个list中有数百万个元素，在头部或尾部添加一个元素的操作，其时间复杂度也是常数级别的。用LPUSH命令在十个元素的list头部添加新元素，和在千万元素list头部添加新元素的速度相同。那么，坏消息是什么？在数组实现的list中利用索引访问元素的速度极快，而同样的操作在linked list实现的list上没有那么快。如果快速访问集合元素很重要，建议使用可排序集合(sorted sets)。可排序集合我们会随后介绍。

1. list入门：
```
127.0.0.1:6379> rpush mylist a
(integer) 1
127.0.0.1:6379> rpush mylist b
(integer) 2
127.0.0.1:6379> lpush mylist first
(integer) 3
```
List命令集
>1. rpush在list的右边添加一个元素（尾部）
>2. lpush在list的左边添加一个元素（头部）
>3. push操作返回的是list内元素的数量
>4. lrange命令可以从list中取出一定范围的元素（:LRANGE带有两个索引，一定范围的第一个和最后一个元素。这两个索引都可以为负来告知Redis从尾部开始计数，因此-1表示最后一个元素，-2表示list中的倒数第二个元素，以此类推。）
>5. lpop/rpop在左边和右边删除元素
>6. list还有阻塞操作，适用于当list为空时，很多访问是无效的，为了减少对Redis的访问压力，redis提供了阻塞式访问 BRPOP 和 BLPOP 命令。 消费者可以在获取数据时指定如果数据不存在阻塞的时间，如果在时限内获得数据则立即返回，如果超时还没有数据则返回null, 0表示一直阻塞。 同时redis还会为所有阻塞的消费者以先后顺序排队。
##### 1.3 Sets
Redis Set是string类型的无序集合。集合成员是唯一的，这就意味着集合中不能出现重复的数据，集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是O(1)。
sadd命令可以把新的元素添加到set中，对set也可以做一些其他的操作，如测试一个元素是否存在、对不同的set取交集、并集或差等。

```
127.0.0.1:6379> sadd myset 1 2 3 4
(integer) 4
127.0.0.1:6379> smembers myset
1) "1"
2) "2"
3) "3"
4) "4"
127.0.0.1:6379>

```

```
127.0.0.1:6379> smembers myset
1) "1"
2) "2"
3) "3"
4) "4"
5) "6"
6) "9"

srem
127.0.0.1:6379> srem myset 5
(integer) 0
127.0.0.1:6379> srem myset 6
(integer) 1

smembers
127.0.0.1:6379> smembers myset
1) "1"
2) "2"
3) "3"
4) "4"
5) "9"

spop
127.0.0.1:6379> spop myset
"2"

srandmember
127.0.0.1:6379> srandmember myset 4
1) "1"
2) "3"
3) "4"
4) "9"
127.0.0.1:6379>

sadd
127.0.0.1:6379> sadd myset how are u i am fine
(integer) 6
127.0.0.1:6379> smembers myset
1) "u"
2) "i"
3) "how"
4) "fine"
5) "are"
6) "am"
127.0.0.1:6379>


sdiff
127.0.0.1:6379> smembers myset
1) "fine"
2) "are"
3) "u"
4) "am"
5) "i"
6) "how"
127.0.0.1:6379> smembers myset1
1) "fine"
2) "3"
3) "2"
4) "4"
5) "1"
127.0.0.1:6379> sdiff myset myset1
1) "are"
2) "am"
3) "u"
4) "i"
5) "how"
127.0.0.1:6379>
```
Set命令集：
>1. spop myset：移除并返回集合中的一个随机元素
>2. srandmember myset count：随机返回集合中count个数
>3. smembers myset:返回集合中的所有元素
>4. scard key：获取集合的成员数
>5. sdiff key1 key2：返回给定所有集合的差集
>6. sdiffstore destinationkey key1  key2:返回给定所有集合的交集并存储在destinationkey中
>7. sunion key1 key2：返回所有给定集合的交集
>8. sunionstore destination key1 key2：返回所有给定集合的并集并存储在destination集合中
>9. sismember key member：判断member元素是否是集合key的成员


##### 1.4 Sorted sets(有序集合)
Redis有序集合和集合一样也是String类型元素的集合，且不允许重复的成员。
不同的是有序集合每个元素都会关联一个double类型的分数，redis正是通过分数来为集合中的成员进行从小到大的排序，有序集合中的成员是唯一的，但分数是可以重复的。
集合是通过hashtable（哈希表）实现的，所以添加、删除、查找的复杂度是o(1),集合中最大的成员数2^23-1(4294967295, 每个集合可存储40多亿个成员)

Sorted sets命令集
>1. zadd key score1 member1 score2 member2:向有序集合添加一个或多个成员，或者更新已存在成员的分数
>2. zcard key：获取有序集合的成员数
>3. zcount key min max：计算在有序集合指定分数区间的成员数
>4. zincrby key increment member：有序集合中对指定成员的分数加上增量increment
>5. zinterstore destination numkeys key1 key2...:计算给定的一个或多个有序集的交集并将结果集存储在新的有序集合key中
>6. zunionstore destination numkeys key1 key2... ：计算给定的一个或多个有序集的并集，并存储在新的key中
>7. zrange key start stop (withscores)：通过有序集合指定索引区间内的成员(带分数)
>8. zrangebyscore key min max :返回指定分数区间内的成员
>9. zscore key member：返回有序集合中制定成员的分数




```
zadd
127.0.0.1:6379> zadd salary 2500 jacl
(integer) 1

zrem
127.0.0.1:6379> zrem salary jacl
(integer) 1
127.0.0.1:6379> zadd salary 2500 jack
(integer) 1
127.0.0.1:6379> zadd salary 4000 rose
(integer) 1
127.0.0.1:6379> zadd salary 10000 yolyn
(integer) 1

zrangebyscore
127.0.0.1:6379> zrangebyscore salary -10000 +10000
1) "tom"
2) "jack"
3) "rose"
4) "yolyn"
```


##### 1.5 Hash（哈希）
Hash命令集
>1. hdel key field1 field2 ：删除一个或多个哈希表字段
>2. hset key field value：将哈希表 key 中的字段 field 的值设为 value
>3. hmset key field1 value1 field2 value2... :同时将多个 field-value (域-值)对设置到哈希表 key 中。
>4. hget key field ：获取存储在哈希表中指定字段的值
>5. hmget key field1 field2：获取所有给定字段的值
>6. hgetall key：获取在哈希表中指定 key 的所有字段和值
>7. hsetnx key field value :只有在字段 field 不存在时，设置哈希表字段的值。
>8. hvals key:获取哈希表中所有值
>9. hkeys key:获取所有哈希表中的字段
>10. hlen key:获取哈希表中字段的数量




```
hmset
127.0.0.1:6379> hmset yolyn fullname houyunlin sex male age 20
OK

hgetall
127.0.0.1:6379> hgetall yolyn
1) "fullname"
2) "houyunlin"
3) "sex"
4) "male"
5) "age"
6) "20"
127.0.0.1:6379>

hvals
127.0.0.1:6379> hvals yolyn
1) "houyunlin"
2) "male"
3) "20"
127.0.0.1:6379>

hkeys
127.0.0.1:6379> hkeys yolyn
1) "fullname"
2) "sex"
3) "age"

hlen
127.0.0.1:6379> hlen yolyn
(integer) 3
127.0.0.1:6379>
```
### 2. why Redis?
市场常见的nosql有redis、memcached、mongodb（面向文档存储的数据库）
先简单了解其他的非关系型数据库。
##### 2.1 Memcached
##### 2.1.1 Memcached简介：
>Memcached 是一个高性能的分布式内存对象缓存系统，用于动态Web应用以减轻数据库负载。它通过在内存中缓存数据和对象来减少读取数据库的次数，从而提供动态、数据库驱动网站的速度，现在已被LiveJournal、hatena、Facebook、Vox、LiveJournal等公司所使用。

##### 2.1.2 Memcached工作方式
> 许多Web应用都将数据保存到 RDBMS中，应用服务器从中读取数据并在浏览器中显示。 但随着数据量的增大、访问的集中，就会出现RDBMS的负担加重、数据库响应恶化、 网站显示延迟等重大影响。Memcached是高性能的分布式内存缓存服务器,通过缓存数据库查询结果，减少数据库访问次数，以提高动态Web等应用的速度、 提高可扩展性。下图展示了memcache与数据库端协同工作情况

![image](https://note.youdao.com/yws/api/personal/file/DC223E4237204D3896EC20BD873B662F?method=download&shareKey=cd2264811a0884e615281c73d5b25bbf)



 其中的过程是这样的：   
1.检查用户请求的数据是缓存中是否有存在，如果有存在的话，只需要直接把请求的数据返回，无需查询数据库。   
2.如果请求的数据在缓存中找不到，这时候再去查询数据库。返回请求数据的同时，把数据存储到缓存中一份。   
3.保持缓存的“新鲜性”，每当数据发生变化的时候（比如，数据有被修改，或被删除的情况下），要同步的更新缓存信息，确保用户不会在缓存取到旧的数据。   

##### 2.1.3 Memcached作为高速运行的分布式缓存服务器的特点
1. 协议简单（tcp/udp）
2. 基于libevent事件处理(一个用C语言编写的、轻量级的开源高性能事件通知库)
3. 内置内存存储方式
4. Memcached不相互通信的分布式(图)
5. Memcached的分布式不是在服务器端实现的，而是在客户端应用中实现的，即通过内置算法([一致性哈希](https://www.cnblogs.com/lpfuture/p/5796398.html))制定目标数据的节点

##### 2.2 Redis支持的数据类型比Memcached多


##### 2.3 过期策略   
过期策略–memcache在set时就指定，例如set key1008,即永不过期。Redis可以通过例如expire 设定，例如expire name 10       
##### 2.4 分布式集群都可以一主多从
##### 2.5 持久化   
存储数据安全–memcache挂掉后，数据没了；redis可以定期保存到磁盘（持久化）
##### 2.6 灾难恢复   
灾难恢复–memcache挂掉后，数据不可恢复; redis数据丢失后可以通过aof恢复




### 3. Redis适用于哪些场景    
1. 缓存       
合理的利用缓存不仅能够提升网站访问速度，还能大大降低数据库的压力。Redis提供了键过期功能，也提供了灵活的键淘汰策略，所以，现在Redis用在缓存的场合非常多。   
2. 计数器    
什么是计数器，如电商网站商品的浏览量、视频网站视频的播放数等。为了保证数据实时效，每次浏览都得给+1，并发量高时如果每次都请求数据库操作无疑是种挑战和压力。Redis提供的incr命令来实现计数器功能，内存操作，性能非常好，非常适用于这些计数场景。     
3. 分布式session   
集群模式下，在应用不多的情况下一般使用容器自带的session复制功能就能满足，当应用增多相对复杂的系统中，一般都会搭建以Redis等内存数据库为中心的session服务，session不再由容器管理，而是由session服务及内存数据库管理。      
4. 分布式锁     
在很多互联网公司中都使用了分布式技术，分布式技术带来的技术挑战是对同一个资源的并发访问，如全局ID、减库存、秒杀等场景，并发量不大的场景可以使用数据库的悲观锁、乐观锁来实现，但在并发量高的场合中，利用数据库锁来控制资源的并发访问是不太理想的，大大影响了数据库的性能。    
5. Message Queue    
消息队列是大型网站必用中间件，如ActiveMQ、RabbitMQ、Kafka等流行的消息队列中间件，主要用于业务解耦、流量削峰及异步处理实时性低的业务。Redis提供了发布/订阅及阻塞队列功能，能实现一个简单的消息队列系统。另外，这个不能和专业的消息中间件相比。   

### 4. Redis其他特性      
1. [RDB&AOF](http://note.youdao.com/noteshare?id=82894bc0665291b754183dc83e3372c4&sub=C77CA4D13A094BA591D66E6586185F58)      
2. 数据淘汰策略    
```

# MAXMEMORY POLICY: how Redis will select what to remove when maxmemory
# is reached? You can select among five behavior:
#
# volatile-lru -> remove the key with an expire set using an LRU algorithm
# allkeys-lru -> remove any key accordingly to the LRU algorithm
# volatile-random -> remove a random key with an expire set
# allkeys->random -> remove a random key, any key
# volatile-ttl -> remove the key with the nearest expire time (minor TTL)
# noeviction -> don't expire at all, just return an error on write operations
#
# Note: with all the kind of policies, Redis will return an error on write
#       operations, when there are not suitable keys for eviction.
#
#       At the date of writing this commands are: set setnx setex append
#       incr decr rpush lpush rpushx lpushx linsert lset rpoplpush sadd
#       sinter sinterstore sunion sunionstore sdiff sdiffstore zadd zincrby
#       zunionstore zinterstore hset hsetnx hmset hincrby incrby decrby
#       getset mset msetnx exec sort
#
# The default is:
#
# maxmemory-policy volatile-lru
```

### 5. Redis集群方案
1. [Redis集群方案](http://note.youdao.com/noteshare?id=dca265ecca4c2e456479fe6aa4cae081&sub=C832B1AF069C4B8FB9E5E79E3513C41F)
