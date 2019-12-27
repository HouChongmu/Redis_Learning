### Redis之RDB&AOF   
#### 1. RDB
RDB持久化方式是通过快照（snapshotting）完成的，当符合一定条件时，redis会自动将内存中所有的数据以二进制方式生成一份副本并存储在硬盘上，当redis重启时，并且AOF持久化未开启，redis会读取RDB持久化生成的二进制文件（即dump.rdb,可以通过dbfilename修改）进行数据恢复，对于持久化信息可以通过命令“info persistence”查看    
##### 快照文件存储位置：     
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
##### 快照触发条件    
##### 1. 客户端执行命令save和bgsave会生成快照         
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

介绍一下上述过程    
1. 客户端执行bgsave时，redis主进程收到指令并判断此时是否在执行bgrewriteaof（AOF文件重写）如果此时正好在执行，那么bgsave直接返回，不fork子进程，如果没有执行bgrewriteaof，则进入下一个阶段      
2. 主进程调用fork方法创建子进程，在创建过程中redis主进程阻塞，所以不能响应客户端请求；    
3. 子进程创建完成以后，bgsave命令返回“Background saving started”，此时标志着redis可以响应客户端请求了；    
4. 子经常根据主进程的内存副本创建临时快照文件，当快照文件完成以后对原快照文件进行替换；    
5. 子进程发送信号给redis主进程完成快照操作，主进程更新统计信息（info Persistence可查看）,子进程退出；



##### 2. 根据配置文件save m n规则进行自动快照        
save m n规则说明：在指定的m秒内，redis中有n个键发生改变，则自动触发bgsave。该规则默认也在redis.conf中进行了配置，并且可组合使用，满足其中一个规则，则触发bgsave      
以save 900 1为例，表明当900秒内至少有一个键发生改变时候，redis触发bgsave操作。
##### 3. 主从复制时，从库全量复制同步主库数据，此时主库会执行bgsave命令进行快照        
redis主从复制中，从节点执行全量操作，主节点会执行bgsave命令，并将rdb文件发送给从节点
##### 4. 客户端执行数据库清空命令flushall时，触发快照           
flushall命令用于清空数据库，请慎用，当我们使用了则表明我们需要对数据进行清空，那redis当然需要对快照文件也进行清空，所以会触发bgsave。
##### 5. 客户端执行shutdown关闭redis时，触发快照    
redis在关闭前处于安全角度将所有数据全部保存下来，以便下次启动时恢复     
我这里手动shutdown后再启动，跟一下日志    
这里可以在启动的时候指定配置：redis-server.exe D:\DevelopSoftwareLocation\Redis-x64-3.0.504\redis.windows.conf   
在conf里面设置了logfile路径
```
[2332] 25 Dec 17:23:59.194 # User requested shutdown...
[2332] 25 Dec 17:23:59.194 * Saving the final RDB snapshot before exiting.
[2332] 25 Dec 17:23:59.206 * DB saved on disk
[2332] 25 Dec 17:23:59.206 # Redis is now ready to exit, bye bye...
                _._                                                  
           _.-``__ ''-._                                             
      _.-``    `.  `_.  ''-._           Redis 3.0.504 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._                                   
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 17636
  `-._    `-._  `-./  _.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |           http://redis.io        
  `-._    `-._`-.__.-'_.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |                                  
  `-._    `-._`-.__.-'_.-'    _.-'                                   
      `-._    `-.__.-'    _.-'                                       
          `-._        _.-'                                           
              `-.__.-'                                               

[17636] 25 Dec 17:28:16.978 # Server started, Redis version 3.0.504
[17636] 25 Dec 17:28:16.979 * DB loaded from disk: 0.000 seconds
[17636] 25 Dec 17:28:16.979 * The server is now ready to accept connections on port 6379

```    
##### 故障恢复
可以发现redis-server在被shutdown之前会save the final rdb snapshot，然后保存到磁盘，启动的时候从磁盘中load db   

##### 配置详解    
```
save m n
#配置快照(rdb)促发规则，格式：save <seconds> <changes>
#save 900 1  900秒内至少有1个key被改变则做一次快照
#save 300 10  300秒内至少有300个key被改变则做一次快照
#save 60 10000  60秒内至少有10000个key被改变则做一次快照
#关闭该规则使用svae “” 

dbfilename  dump.rdb
#rdb持久化存储数据库文件名，默认为dump.rdb

stop-write-on-bgsave-error yes 
#yes代表当使用bgsave命令持久化出错时候停止写RDB快照文件,no表明忽略错误继续写文件。

rdbchecksum yes
#在写入文件和读取文件时是否开启rdb文件检查，检查是否有无损坏，如果在启动是检查发现损坏，则停止启动。

dir "/etc/redis"
#数据文件存放目录，rdb快照文件和aof文件都会存放至该目录，请确保有写权限

rdbcompression yes
#是否开启RDB文件压缩，该功能可以节约磁盘空间
```



#### 2. AOF持久化    
当redis存储非临时数据时，为了降低redis故障而引起的数据丢失，redis提供了AOF(Append Only File)持久化，从单词意思讲，将命令追加到文件。AOF可以将Redis执行的每一条写命令追加到磁盘文件(appendonly.aof)中,在redis启动时候优先选择从AOF文件恢复数据。由于每一次的写操作，redis都会记录到文件中，所以开启AOF持久化会对性能有一定的影响，但是大部分情况下这个影响是可以接受的，我们可以使用读写速率高的硬盘提高AOF性能。与RDB持久化相比，AOF持久化数据丢失更少，其消耗内存更少(RDB方式执行bgsave会有内存拷贝)      
#####  配置参数    
```
auto-aof-rewrite-min-size 64mb
#AOF文件最小重写大小，只有当AOF文件大小大于该值时候才可能重写,4.0默认配置64mb。

auto-aof-rewrite-percentage  100
#当前AOF文件大小和最后一次重写后的大小之间的比率等于或者等于指定的增长百分比，如100代表当前AOF文件是上次重写的两倍时候才重写。

appendfsync everysec
#no：不使用fsync方法同步，而是交给操作系统write函数去执行同步操作，在linux操作系统中大约每30秒刷一次缓冲。这种情况下，缓冲区数据同步不可控，并且在大量的写操作下，aof_buf缓冲区会堆积会越来越严重，一旦redis出现故障，数据
#always：表示每次有写操作都调用fsync方法强制内核将数据写入到aof文件。这种情况下由于每次写命令都写到了文件中, 虽然数据比较安全，但是因为每次写操作都会同步到AOF文件中，所以在性能上会有影响，同时由于频繁的IO操作，硬盘的使用寿命会降低。
#everysec：数据将使用调用操作系统write写入文件，并使用fsync每秒一次从内核刷新到磁盘。 这是折中的方案，兼顾性能和数据安全，所以redis默认推荐使用该配置。

aof-load-truncated yes
#当redis突然运行崩溃时，会出现aof文件被截断的情况，Redis可以在发生这种情况时退出并加载错误，以下选项控制此行为。
#如果aof-load-truncated设置为yes，则加载截断的AOF文件，Redis服务器启动发出日志以通知用户该事件。
#如果该选项设置为no，则服务将中止并显示错误并停止启动。当该选项设置为no时，用户需要在重启之前使用“redis-check-aof”实用程序修复AOF文件在进行启动。

appendonly no 
#yes开启AOF，no关闭AOF

appendfilename appendonly.aof
#指定AOF文件名，4.0无法通过config set 设置，只能通过修改配置文件设置。

dir /etc/redis
#RDB文件和AOF文件存放目录
```
##### 2.1 开启AOF   
默认情况下，redis是关闭了AOF持久化，开启AOF可以通过配置appendonly为yes，可以直接使用命令行config set appendonly yes，再用config rewrite同步到配置文件。好处是，不用重启redis，配置实时生效。     
```
127.0.0.1:6379> config set loglevel verbose
OK
(1.53s)
127.0.0.1:6379> config set appendonly yes
OK
(1.60s)
127.0.0.1:6379> config rewrite
OK
127.0.0.1:6379>
```   
##### 2.2 AOF持久化过程      
1. 追加写入    
redis将每一条写命令以redis通讯协议添加至缓冲区aof_buf,这样的好处在于在大量写请求情况下，采用缓冲区暂存一部分命令随后根据策略一次性写入磁盘，这样可以减少磁盘的I/O次数，提高性能。    
2. 同步命令到磁盘      
当写命令写入aof_buf缓冲区后，redis会将缓冲区的命令写入到文件，redis提供了三种同步策略，由配置参数appendfsync决定，下面是每个策略所对应的含义：   
- 1. no：不使用fsync方法同步，而是交给操作系统write函数去执行同步操作，在linux操作系统中大约每30秒刷一次缓冲。这种情况下，缓冲区数据同步不可控，并且在大量的写操作下，aof_buf缓冲区会堆积会越来越严重，一旦redis出现故障，数据丢失严重。    
- 2. always：表示每次有写操作都调用fsync方法强制内核将数据写入到aof文件。这种情况下由于每次写命令都写到了文件中,虽然数据比较安全，但是因为每次写操作都会同步到AOF文件中，所以在性能上会有影响，同时由于频繁的IO操作，硬盘的使用寿命会降低。   
- 3. everysec：数据将使用调用操作系统write写入文件，并使用fsync每秒一次从内核刷新到磁盘。 这是折中的方案，兼顾性能和数据安全，所以redis默认推荐使用该配置。   
3. 文件重写（bgrewriteaof）    
当开启的AOF时，随着时间推移，AOF文件会越来越大,当然redis也对AOF文件进行了优化，即触发AOF文件重写条件时候，redis将使用bgrewriteaof对AOF文件进行重写。这样的好处在于减少AOF文件大小，同时有利于数据的恢复。      
##### 为什么重写？    
假如先后执行了“set age 1;set age 2;set age 3”开启了AOF，那这时候AOF文件会记录三条命令，这显然是不合理的，因为文件中应该只保留set age 3;这条命令，否则会有很多冗余的数据，Redis考虑到了这点，引入可AOF重写机制，重写策略如下：    
* 重复或无效的命令不写入文件   
* 过期的数据不再写入文件   
* 多条命令合并写入（当多个命令能合并一条命令时候会对其优化合并作为一个命令写入，例如“RPUSH list1 a RPUSH list1 b" 合并为“RPUSH list1 a b” ）     

4. 重写触发条件（区分重写策略）   
     
手动触发：客户端执行bgrewriteaof命令   
自动触发：每次当serverCron函数执行时，它会检查以下条件是否全部满足，如果全部满足的话，就触发自动的AOF重写操作，自动触发通过以下两个配置协作生效再加两个判断     
* auto-aof-rewrite-min-size：AOF文件最小重写大小，只有当AOF文件大小大于该值时候才可能重写，4.0默认配置是64M
* auto-aof-rewrite-percentage：当前AOF文件大小和最后一次重写后的大小之间的比率等于或等于指定的增长百分比，如100代表当前AOF文件是上次重写文件的两倍的时候才重写       
* 没有bgsave命令在执行   
* 没有bgrewriteaof在执行  



redis在开启AOF功能开启的情况下，会维持以下三个变量。    
aof_current_size:记录当前AOF文件大小的变量。
aof_rewrite_base_size:记录最后一次AOF重写之后，AOF文件大小的变量。
aof_rewrite_perc:增长百分比变量。    



##### 2.3 重写过程：      
<img width="500" src="https://note.youdao.com/yws/api/personal/file/ED968B28C86A4252BB0650054ABE3689?method=download&shareKey=94be758a7b130061c7fb6f9d03b98eaf"/>

1. 主进程接收到bgrewriteaof命令首先判断是否有bgsave/bgrewriteaof正在执行，若有，则等待执行完再处理bgrewriteaof     
2. 通过上一步后，fork出一个子进程，这段时间主进程是阻塞的，子进程准备完毕后再开始处理新来的命令。     
- 2.1 子进程通过内存快照开始重写AOF文件
- 2.2 接收aof_rewrite_buf缓冲区内容写入新AOF文件
3. 在子进程重写AOF文件的同时，主进程会将接收到的命令写到aof_rewrite_buf和aof_buf缓冲区，其中aof_rewrite_buf是重写缓冲区，aof_buf是aof存放区，重写缓冲区内容会同步到新的AOF文件    
4. 重写完成后，子进程通知主进程，主进程更新persistence info并用新生成的AOF文件替换旧的AOF文件，即AOF重写完成      



AOF是基于redis通讯协议，将命令以纯文本的方式写到文件中
##### Redis 协议     

```
首先Redis是以行来划分，每行以\r\n行结束。每一行都有一个消息头，消息头共分为5种分别如下:

(+) 表示一个正确的状态信息，具体信息是当前行+后面的字符。

(-)  表示一个错误信息，具体信息是当前行－后面的字符。

(*) 表示消息体总共有多少行，不包括当前行,*后面是具体的行数。

($) 表示下一行数据长度，不包括换行符长度\r\n,$后面则是对应的长度的数据。

(:) 表示返回一个数值，：后面是相应的数字节符。
```
#### 3. RDB&AOF混合写     
