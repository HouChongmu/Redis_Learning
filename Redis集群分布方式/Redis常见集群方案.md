### Redis集群方案
#### 1. twemproxy
##### 1.1 何为twemproxy
Twemproxy是一种代理分片机制，由Twitter开源。Twemproxy作为代理，可接受来自多个程序的访问，按照路由规则，转发给后台的各个Redis或memcached服务器，再原路返回，一般路由规则是按照一致性哈希设计，该方案很好的解决了单个Redis或memcached实例承载能力的问题。Twemproxy本身也是单点，需要用Keepalived做高可用方案，可以使用多台服务器来水平扩张redis或memcached服务，可以有效的避免单点故障问题。
##### 1.2 应用于Redis
使用方法和普通 Redis 无任何区别，设置好它下属的多个Redis实例后，使用时在本需要连接Redis的地方改为 连接twemproxy， 它会以一个代理的身份接收请求并使用一致性hash算法，将请求转接到具体Redis，将结果再返回 twemproxy。

##### 1.3 twemproxy存在的问题
1. twemproxy 自身单端口实例的压力
2. 使用一致性 hash 后， 对Redis节点数量改变时候的计算值的改变，数据无法自动移动到新的节点。
3. 并不是支持所有redis命令


#### 2 codis
##### 2.1 何为Codis
Codis 是一个分布式 Redis 解决方案, 对于上层的应用来说, 连接到 Codis Proxy 和连接原生的 Redis Server 没有明显的区别 (不支持的命令列表), 上层应用可以像使用单机的 Redis 一样使用, Codis 底层会处理请求的转发, 不停机的数据迁移等工作, 所有后边的一切事情, 对于前面的客户端来说是透明的, 可以简单的认为后边连接的是一个内存无限大的 Redis 服务。


##### 2.2 组成部分
<img width="500" src="https://note.youdao.com/yws/api/personal/file/48E4A799421A489EB97E5183245A7DBA?method=download&shareKey=e2dff3ab22c36e4e8566cb889e633135"/>


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

##### 2.3 Codis之slot（槽点）
1. 因为Codis代理了Redis服务，所以我们发起的请求，并不是请求到Redis-server所有的机器上，而是到Codis机器上，Codis机器再根据一定的路由规则进行分发，最终请求到Redis-server的机器上，也就是说我们几个get请求，可能会分布到多台机器上。    

<img width="500" src="https://note.youdao.com/yws/api/personal/file/81528B619A784DEDBE365A8FE064ADB7?method=download&shareKey=4da594bdb38974fe3d00bf2b37cbb5d2"/>   


2. 很显然，这样的Codis也会存在单点问题，由于Codis是一个无状态服务（该服务运行的实例不会在本地存储需要持久化的数据，并且多个实例对于同一个请求响应的结果是完全一致的），所以我们可以同时部署多个Codis实例。   

<img width="500" src="https://note.youdao.com/yws/api/personal/file/8A8E53AB608E4E5EAD5EE5AD0A70625B?method=download&shareKey=271a12a96f5dd18132af358dbf1c8ad1"/>      

3. 那Codis是如何将key分配到不同的机器呢    

<img width="500" src="https://note.youdao.com/yws/api/personal/file/4790D9B2322A4A0595052C66E37542B0?method=download&shareKey=b630085b92c9eeb712c66db8b01c7f7f"/>     


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

<img width="500" src="https://note.youdao.com/yws/api/personal/file/DCA7798E85174EC0ADEFD4BF59E594D7?method=download&shareKey=52b84aebc470d5342a4e5c61f1c4bb97"/> 


6. codis proxy的异常处理：codis-ha
codis这里摒弃了哨兵机制，另起了一个进程为codis-ha，codis-ha实时监控proxy的运行状态，如果有异常就会干掉，利用k8s(容器的编排工具)的特性，会自动拉起一个proxy，但是codis-ha在codis整个结构中是没法直接操作代理和服务的，因为所有代理和服务的操作都要经过dashboard处理，所以部署的时候一般将codis-ha和dashboard部署在同一个节点上（可以分布在不同的pod（集群调度的最小单元，只能运行在一个主机）上）
同时，codis-ha也会检测各server group的redis成员能都被ping通，如果不能被PING不通，且其为某组的MASTER角色，则将同组某SLAVE提升为MASTER角色；如果其为某组的SLAVE，则无需做任何处理。       

<img width="500" src="https://note.youdao.com/yws/api/personal/file/255081D28D8A4DAF86CFBD337DD3801E?method=download&shareKey=87a13bd497b7db7766029e4eda1618fa"/> 

7. 集群管理界面
Codis-fe，操作dashboard

8. 最后redis客户端，通过codis-proxy来访问redis      


<img width="500" src="https://note.youdao.com/yws/api/personal/file/0CCBAD16B77741E48D2576B4994678D7?method=download&shareKey=3c4d15110b0a9a7bf38fc31ce6556cb6"/> 



#### 3 Redis Cluster
它和codis不太一样，Codis它是通过代理分片的，但是Redis Cluster是去中心化的没有代理，所以只能通过客户端分片，它分片的槽数跟Codis不太一样，Codis是1024个，而Redis cluster有16384个，槽跟节点的映射关系保存在每个节点上，每个节点每秒钟会ping十次其他几个最久没通信的节点，其他节点也是一样的原理互相PING ，PING的时候一个是判断其他节点有没有问题，另一个是顺便交换一下当前集群的节点信息、包括槽与节点映射的关系等。客户端操作key的时候先通过分片算法算出所属的槽，然后随机找一个服务端请求。     

<img width="500" src="https://note.youdao.com/yws/api/personal/file/1FA5168E58A646C0B98D2A2BB3C90718?method=download&shareKey=20a3fc4e210aeaeb0fe5e06fc3d3cc9e"/> 

但是可能这个槽并不归随机找的这个节点管，节点如果发现不归自己管，就会返回一个MOVED ERROR通知，引导客户端去正确的节点访问，这个时候客户端就会去正确的节点操作数据。
>规则：  
>1.每个节点通过通信都会共享Redis Cluster中槽和集群中对应节点的关系  
>2.客户端向Redis Cluster的任意节点发送命令，接收命令的节点会根据CRC16规则进行hash运算与16383取余，计算自己的槽和对应节点  
>3.如果保存数据的槽被分配给当前节点，则去槽中执行命令，并把命令执行结果返回给客户端  
>4.如果保存数据的槽不在当前节点的管理范围内，则向客户端返回moved重定向异常  
>5.客户端接收到节点返回的结果，如果是moved异常，则从moved异常中获取目标节点的信息  
>6.客户端向目标节点发送命令，获取命令执行结果   