### 1. Redis基本使用
### 2. Redis特点
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
##### 2.1. Stirng
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

1. INCR 命令将字符串值解析成整型，将其加一，最后将结果保存为新的字符串值。
类似的命令有INCRBY, DECR 和 DECRBY。实际上他们在内部就是同一个命令，只是看上去有点儿不同。
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

INCR是原子操作意味着什么呢？就是说即使多个客户端对同一个key发出INCR命令，也决不会导致竞争的情况。例如如下情况永远不可能发生：『客户端1和客户端2同时读出“10”，他们俩都对其加到11，然后将新值设置为11』。最终的值一定是12，read-increment-set操作完成时，其他客户端不会在同一时间执行任何命令。

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


##### 2.2 Lists
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
##### 2.3 Sets
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


##### 2.4 Sorted sets(有序集合)
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


##### 2.5 Hash（哈希）
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
### 3. why Redis?
市场常见的nosql有redis、memcached、mongodb（面向文档存储的数据库）
先简单了解其他的非关系型数据库。
#### 3.1 Memcached
##### 3.1.1 Memcached简介：
>Memcached 是一个高性能的分布式内存对象缓存系统，用于动态Web应用以减轻数据库负载。它通过在内存中缓存数据和对象来减少读取数据库的次数，从而提供动态、数据库驱动网站的速度，现在已被LiveJournal、hatena、Facebook、Vox、LiveJournal等公司所使用。

##### 3.1.2 Memcached工作方式
> 许多Web应用都将数据保存到 RDBMS中，应用服务器从中读取数据并在浏览器中显示。 但随着数据量的增大、访问的集中，就会出现RDBMS的负担加重、数据库响应恶化、 网站显示延迟等重大影响。Memcached是高性能的分布式内存缓存服务器,通过缓存数据库查询结果，减少数据库访问次数，以提高动态Web等应用的速度、 提高可扩展性。下图展示了memcache与数据库端协同工作情况

![image](https://note.youdao.com/yws/api/personal/file/DC223E4237204D3896EC20BD873B662F?method=download&shareKey=cd2264811a0884e615281c73d5b25bbf)



 其中的过程是这样的：
1.检查用户请求的数据是缓存中是否有存在，如果有存在的话，只需要直接把请求的数据返回，无需查询数据库。
2.如果请求的数据在缓存中找不到，这时候再去查询数据库。返回请求数据的同时，把数据存储到缓存中一份。
3.保持缓存的“新鲜性”，每当数据发生变化的时候（比如，数据有被修改，或被删除的情况下），要同步的更新缓存信息，确保用户不会在缓存取到旧的数据。

##### 3.1.3 Memcached作为高速运行的分布式缓存服务器的特点
1. 协议简单（tcp/udp）
2. 基于libevent事件处理(一个用C语言编写的、轻量级的开源高性能事件通知库)
3. 内置内存存储方式
4. Memcached不相互通信的分布式(图)
5. Memcached的分布式不是在服务器端实现的，而是在客户端应用中实现的，即通过内置算法([一致性哈希](https://www.cnblogs.com/lpfuture/p/5796398.html))制定目标数据的节点

#### 3.2 Redis支持的数据类型比Memcached多
##### 3.1.2     


3.3 过期策略–memcache在set时就指定，例如set key1 0 0 8,即永不过期。Redis可以通过例如expire 设定，例如expire name 10   
3.4 分布式–设定memcache集群，利用magent做一主多从;redis可以做一主多从。都可以一主一从
3.5 存储数据安全–memcache挂掉后，数据没了；redis可以定期保存到磁盘（持久化）
3.6 灾难恢复–memcache挂掉后，数据不可恢复; redis数据丢失后可以通过aof恢复




### 4. Redis适用于哪些场景    
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

### 5. Redis其他特性      

#### 5.1 RDB&AOF   
##### 5.1.1 RDB
RDB持久化方式是通过快照（snapshotting）完成的，当符合一定条件时，redis会自动将内存中所有的数据以二进制方式生成一份副本并存储在硬盘上，当redis重启时，并且AOF持久化未开启，redis会读取RDB持久化生成的二进制文件（即dump.rdb,可以通过dbfilename修改）进行数据恢复，对于持久化信息可以通过命令“info persistence”查看    
快照文件存储位置：     
RDB快照文件存储位置有dir配置参数指明，文件名由dbfilename指定          
```
127.0.0.1:6379> config get dir
1) "dir"
2) "C:\\Users\\houyl"
127.0.0.1:6379> config get dbfilename
1) "dbfilename"
2) "dump.rdb"
127.0.0.1:6379> config set dbfilename redis_dump.rdb
OK
127.0.0.1:6379> config get dbfilename
1) "dbfilename"
2) "redis_dump.rdb"
127.0.0.1:6379>
```
快照触发条件    
1. 客户端执行命令save和bgsave会生成快照         
>客户端执行save命令，该命令强制redis执行快照，这时候redis处于阻塞状态，不会响应任何其他客户端发来的请求，直到RDB快照文件执行完毕，所以请慎用。       
save:  
```
127.0.0.1:6379> info persistence
# Persistence
loading:0
rdb_changes_since_last_save:0
rdb_bgsave_in_progress:0
rdb_last_save_time:1577252161
rdb_last_bgsave_status:ok
rdb_last_bgsave_time_sec:-1
rdb_current_bgsave_time_sec:-1
aof_enabled:0
aof_rewrite_in_progress:0
aof_rewrite_scheduled:0
aof_last_rewrite_time_sec:-1
aof_current_rewrite_time_sec:-1
aof_last_bgrewrite_status:ok
aof_last_write_status:ok
127.0.0.1:6379> save
OK
127.0.0.1:6379> info persistence
# Persistence
loading:0
rdb_changes_since_last_save:0
rdb_bgsave_in_progress:0
rdb_last_save_time:1577252506
rdb_last_bgsave_status:ok
rdb_last_bgsave_time_sec:-1
rdb_current_bgsave_time_sec:-1
aof_enabled:0
aof_rewrite_in_progress:0
aof_rewrite_scheduled:0
aof_last_rewrite_time_sec:-1
aof_current_rewrite_time_sec:-1
aof_last_bgrewrite_status:ok
aof_last_write_status:ok
127.0.0.1:6379>
```   
bgsave:    
bgsave命令可以理解为background save，当执行了bgsave时，redis会fork出一个子进程来执行快照生成操作，并且redis fork这个子进程的过程中redis是阻塞的，这段时间不会响应客户端的请求，当子进程创建完成之后redis开始响应客户端。其实redis自动快照也是用bgsave来完成的     
<img width="300" src="https://note.youdao.com/yws/api/personal/file/EC52D691F1264BC8BB2BE51C36997F97?method=download&shareKey=0066db09a8a3b580a76976d588221222"/>


2. 根据配置文件save m n规则进行自动快照        

3. 主从复制时，从库全量复制同步主库数据，此时主库会执行bgsave命令进行快照      
4. 客户端执行数据库清空命令flushall时，触发快照        
5. 客户端执行shutdown关闭redis时，触发快照

### 6. Redis集群方案
#### 6.1 twemproxy
##### 6.1.1 何为twemproxy
Twemproxy是一种代理分片机制，由Twitter开源。Twemproxy作为代理，可接受来自多个程序的访问，按照路由规则，转发给后台的各个Redis或memcached服务器，再原路返回，一般路由规则是按照一致性哈希设计，该方案很好的解决了单个Redis或memcached实例承载能力的问题。Twemproxy本身也是单点，需要用Keepalived做高可用方案，可以使用多台服务器来水平扩张redis或memcached服务，可以有效的避免单点故障问题。
##### 6.1.2 应用于Redis
使用方法和普通 Redis 无任何区别，设置好它下属的多个Redis实例后，使用时在本需要连接Redis的地方改为 连接twemproxy， 它会以一个代理的身份接收请求并使用一致性hash算法，将请求转接到具体Redis，将结果再返回 twemproxy。

##### 6.1.3 twemproxy存在的问题
1. twemproxy 自身单端口实例的压力
2. 使用一致性 hash 后， 对Redis节点数量改变时候的计算值的改变，数据无法自动移动到新的节点。
3. 并不是支持所有redis命令



#### 6.2 codis
##### 6.2.1 何为Codis
Codis 是一个分布式 Redis 解决方案, 对于上层的应用来说, 连接到 Codis Proxy 和连接原生的 Redis Server 没有明显的区别 (不支持的命令列表), 上层应用可以像使用单机的 Redis 一样使用, Codis 底层会处理请求的转发, 不停机的数据迁移等工作, 所有后边的一切事情, 对于前面的客户端来说是透明的, 可以简单的认为后边连接的是一个内存无限大的 Redis 服务。


##### 6.2.2 组成部分
![image](https://note.youdao.com/yws/api/personal/file/48E4A799421A489EB97E5183245A7DBA?method=download&shareKey=e2dff3ab22c36e4e8566cb889e633135)


1. Codis Proxy
处理客户端请求，支持Redis协议，因此客户端访问Codis与访问原生Redis没有什么区别
2. Codis Dashboard（codis-config）
Codis的管理工具，支持添加/删除Proxy节点，发起数据迁移等操作。codis-config本身还自带了一个http-server，会启动一个dashboard，用户可以直接在浏览器上观察Codis集群的运行状态
3. Codis Redis（codis-server）
Codis项目维护的一个Redis分支，基于2.8.21开发，加入slot（槽点）的支持和原子的数据迁移指令。
4. ZooKeeper/Etcd
Codis依赖ZooKeeper来存放数据路由表和codis-proxy节点的元信息，codis-confgi发起的命令都会通过ZooKeeper同步到各个存活的codis-proxy
5. codis-ha
通过codis开放的api实现自动切换主从的工具，该工具会在检测到master挂掉的时候将其下线并选择其中一个slave提升为master继续提供服务
>在k8s里面可以设置pod冗余的数量，k8s会严格保证启动的数量与设置的一致，所以只需要一个进程检测proxy的异常，那就是codis-ha。

##### 6.2.3 Codis之slot（槽点）
1. 因为Codis代理了Redis服务，所以我们发起的请求，并不是请求到Redis-server所有的机器上，而是到Codis机器上，Codis机器再根据一定的路由规则进行分发，最终请求到Redis-server的机器上，也就是说我们几个get请求，可能会分布到多台机器上。
![image](https://note.youdao.com/yws/api/personal/file/81528B619A784DEDBE365A8FE064ADB7?method=download&shareKey=4da594bdb38974fe3d00bf2b37cbb5d2)
2. 很显然，这样的Codis也会存在单点问题，由于Codis是一个无状态服务（该服务运行的实例不会在本地存储需要持久化的数据，并且多个实例对于同一个请求响应的结果是完全一致的），所以我们可以同时部署多个Codis实例。
![image](https://note.youdao.com/yws/api/personal/file/8A8E53AB608E4E5EAD5EE5AD0A70625B?method=download&shareKey=271a12a96f5dd18132af358dbf1c8ad1)

3. 那Codis是如何将key分配到不同的机器呢
![image](https://note.youdao.com/yws/api/personal/file/4790D9B2322A4A0595052C66E37542B0?method=download&shareKey=b630085b92c9eeb712c66db8b01c7f7f)
```
//Codis中Key的算法
hash = crc32(command.key)
    slot_index = hash % 1024
    redis = slots[slot_index].redis
    redis.do(command)
```
>在Codis里面，它把所有key都分为1024个槽，每个槽位都对应了一个分组，具体的槽位的分配也可以进行自定义，现在如果有一个key进来，首先要根据CRC32算法，针对key算出32位的哈希值，然后对1024取余，然后就能算出这个key属于哪个槽，确定了槽点之后，根据分组情况确定该key对应哪个Redis分组，然后就能去对应的分组中去处理数据了。
>slot和分组的映射关系就保存在codis proxy中


4. codis proxy
>槽位和分组的映射关系都保存在codis proxy中，但codis proxy本身也存在单点问题，所以需要对proxy做一个集群。

5. proxy同步配置信息
>部署好codis proxy集群之后存在一个问题，槽位的映射关系是保存在proxy里面的，不同的proxy之间是怎么同步映射关系的呢。
>在codis中是使用zookeeper来保存映射关系，由proxy来同步配置信息

![image](https://note.youdao.com/yws/api/personal/file/DCA7798E85174EC0ADEFD4BF59E594D7?method=download&shareKey=52b84aebc470d5342a4e5c61f1c4bb97)

6. codis proxy的异常处理：codis-ha
codis这里摒弃了哨兵机制，另起了一个进程为codis-ha，codis-ha实时监控proxy的运行状态，如果有异常就会干掉，利用k8s(容器的编排工具)的特性，会自动拉起一个proxy，但是codis-ha在codis整个结构中是没法直接操作代理和服务的，因为所有代理和服务的操作都要经过dashboard处理，所以部署的时候一般将codis-ha和dashboard部署在同一个节点上（可以分布在不同的pod（集群调度的最小单元，只能运行在一个主机）上）
同时，codis-ha也会检测各server group的redis成员能都被ping通，如果不能被PING不通，且其为某组的MASTER角色，则将同组某SLAVE提升为MASTER角色；如果其为某组的SLAVE，则无需做任何处理。
![image](https://note.youdao.com/yws/api/personal/file/255081D28D8A4DAF86CFBD337DD3801E?method=download&shareKey=87a13bd497b7db7766029e4eda1618fa)

7. 集群管理界面
Codis-fe，操作dashboard

8. 最后redis客户端，通过codis-proxy来访问redis
![image](https://note.youdao.com/yws/api/personal/file/0CCBAD16B77741E48D2576B4994678D7?method=download&shareKey=3c4d15110b0a9a7bf38fc31ce6556cb6)



#### 6.3 Redis Cluster
它和codis不太一样，Codis它是通过代理分片的，但是Redis Cluster是去中心化的没有代理，所以只能通过客户端分片，它分片的槽数跟Codis不太一样，Codis是1024个，而Redis cluster有16384个，槽跟节点的映射关系保存在每个节点上，每个节点每秒钟会ping十次其他几个最久没通信的节点，其他节点也是一样的原理互相PING ，PING的时候一个是判断其他节点有没有问题，另一个是顺便交换一下当前集群的节点信息、包括槽与节点映射的关系等。客户端操作key的时候先通过分片算法算出所属的槽，然后随机找一个服务端请求。
![image](https://note.youdao.com/yws/api/personal/file/1FA5168E58A646C0B98D2A2BB3C90718?method=download&shareKey=20a3fc4e210aeaeb0fe5e06fc3d3cc9e)

但是可能这个槽并不归随机找的这个节点管，节点如果发现不归自己管，就会返回一个MOVED ERROR通知，引导客户端去正确的节点访问，这个时候客户端就会去正确的节点操作数据。
>规则：  
>1.每个节点通过通信都会共享Redis Cluster中槽和集群中对应节点的关系  
>2.客户端向Redis Cluster的任意节点发送命令，接收命令的节点会根据CRC16规则进行hash运算与16383取余，计算自己的槽和对应节点  
>3.如果保存数据的槽被分配给当前节点，则去槽中执行命令，并把命令执行结果返回给客户端  
>4.如果保存数据的槽不在当前节点的管理范围内，则向客户端返回moved重定向异常  
>5.客户端接收到节点返回的结果，如果是moved异常，则从moved异常中获取目标节点的信息  
>6.客户端向目标节点发送命令，获取命令执行结果    
