
<!-- TOC -->    
- [1.数据类型](#1数据类型)  
- [2.数据结构](#2数据结构)          
    - [2.1 字典](#21-字典)             
        - [2.1.1 数据结构](#211-数据结构)     
        - [2.1.2 哈希算法与冲突处理](#212-哈希算法与冲突处理)       
        - [2.1.3 Rehash](#213-rehash)          
        - [2.1.4 渐进式Rehash](#214-渐进式rehash)   
    - [2.2 SkipList 跳表 (Zset的底层实现)](#22-skiplist-跳表-zset的底层实现)    
        - [2.2.1 数据结构](#221-数据结构)          
        - [2.2.2 查找、插入与删除](#222-查找插入与删除)   
        - [2.2.3 计算排名](#223-计算排名) 
- [3.使用场景](#3使用场景)  
- [4.Key过期机制](#4key过期机制)  
- [5.数据淘汰策略](#5数据淘汰策略)   
    - [5.1Redis缓存淘汰策略：](#51redis缓存淘汰策略)      
    - [5.2 LRU算法](#52-lru算法)           
        - [5.2.1 LRU的常见实现：](#521-lru的常见实现)    
        - [5.2.2 Redis中的LRU](#522-redis中的lru)  
- [6.持久化:RDB和AOF](#6持久化rdb和aof) 
- [7.事务](#7事务)         
    - [7.1ACID特性](#71acid特性)     
    - [7.2 相关命令](#72-相关命令)  
- [8.事件](#8事件)    
    - [8.1 文件事件](#81-文件事件)    
    - [8.2 时间事件](#82-时间事件)       
    - [8.3 Redis 运行流程](#83-redis-运行流程) 
- [9. 主从服务器](#9-主从服务器)        
    - [9.1连接过程](#91连接过程)       
    - [9.2 主从链](#92-主从链)       
    - [9.3 Sentinel 哨兵](#93-sentinel-哨兵)
- [10. 数据分片](#10-数据分片)   
<!-- /TOC -->


# 1.数据类型

| 数据类型 | 可存储的值             | 操作                                                         |
| -------- | ---------------------- | ------------------------------------------------------------ |
| String   | 字符串、整数、浮点数   | 对字符串或者字符串的一部分执行操作<br />对整数和浮点数执行自增或自减操作 |
| List     | 列表                   | 从两端压入、弹出元素<br />返回一个范围内的元素               |
| Set      | 无序集合               | 添加、获取、移除单个元素<br />检查一个元素是否再集合中<br />计算交集、并集、差集<br />从集合中随机获取元素 |
| Hash     | 包含键值对的无序散列表 | 添加、获取、移除单个键值对<br />获取所有键值对<br />检查某个键是否存在 |
| Zset     | 有序集合               | 添加、获取、删除元素<br />根据分值(score)范围或者成员来获取元素<br />计算一个键的排名 |



Zset:

```c
// zset-key下，member1对应的score为728
// zadd 添加
zadd zset-key 728 member1
zadd zset-key 982 member0
// zrange 按范围查看元素，从0到-1为全部, withscores则为一行键一行值
zrange zset-key 0 -1 withscores
// zrangebyscore 按照score的范围获取元素
zrange zset-key 0 800 withscores
// zrem key member1 删除key下 member1 元素
zrem zset-key member1
```



# 2.数据结构



### 2.1 字典

#### 2.1.1 数据结构

Redis中字典采用拉链法来实现。

dict字典结构

```c
typedef struct dict {
    dictType *type;
    void *provdata;
    dictht ht[2];   // 两个hash表，其中ht[0]为正在使用，ht[1]为在rehash时使用。以两个哈希表来实现渐进式rehash.
    long rehashidx; // 正在被rehash的位置. 等于-1时为没有在rehash.
    unsigned long iterators; // 正在运行的迭代器的数量
} dict;
```

dictht结构(哈希表结构):

```c
typedef struct dictht {
    dictEntry ** table; // 存储结点的数组
    unsigned long size;	// 哈希表数组大小
    unsigned long sizemask; // 掩码，一般为size-1。用于将哈希值转换为索引值。
    unsigned long used; // 已使用的结点数量
} dictht;
```

dictEntry(哈希表中的某个节点的结构)：

```c
typedef struct dictEntry {
    void* key;
    union {				// 存储value的联合体
        void* value;	// 自定义类型
        uint64_t u64;   // 无符号整数
        int64_t s64;	// 有符号整数
        double d;		// 浮点数
    } v;
    struct dictEntry* next; // 指向下一个节点
} dictEntry;
```

![img](https://camo.githubusercontent.com/216fba21b84dee3403ec2629aad293ce5e01ee47/68747470733a2f2f75706c6f61642d696d616765732e6a69616e7368752e696f2f75706c6f61645f696d616765732f343434303931342d626238363337346135663265313564362e706e673f696d6167654d6f6772322f6175746f2d6f7269656e742f7374726970253743696d61676556696577322f322f772f31323430)

即字典中含有两个哈希表，每个哈希表包括一个 **指向链表的指针的数组**，每个链表中可以有多个元素。



#### 2.1.2 哈希算法与冲突处理

**哈希算法**

当插入一对键值对到哈希表中时，首先根据 `dict->type->hashFunction(key)` 来计算出key的hash值，再使用hash值和掩码sizemask来进行 `与计算` 得到索引值。

如插入key 为 key0，值为value0的元素时，发生以下计算：

```c
hash = dict->type->hashFunction(key0) 
index = hash & dict->ht[0]->sizemask;
```

假设得出index值为9，redis便在`dict->ht[0]->dictEntry[9]`处加入一个新节点。



**冲突处理**

redis 使用链地址法来解决冲突。

当计算出的index处已经有节点时，redis便将此节点加入到哈希表对应index的链表头部。



#### 2.1.3 Rehash

**rehash的触发条件**

`负载因子 = 哈希表中已保存的元素总数 / 哈希表中链表的个数 `

即 `load_factor = ht[0].used / ht[0].size`

- 当负载因子超过1，则表明哈希表中元素过多，需要`扩容`
  - 如果redis正在做bgsave或bgrewriteaof，为了减少过多的 Copy On Write，load_fatcor超过5时才会强制扩容。
  - 如果redis没有在bgsave或bgrewriteaof，那么当load_factor >= 1时便会扩容
- 当负载因子小于0.1时，便会引发 `收缩`

**rehash步骤**

1. 为字典的ht[1]哈希表分配空间，分配空间的大小取决于ht[0]的used的大小。
   - 执行的为扩容操作：ht[1].size 为 第一个大于等于 **ht[0].used*2 **的 2的n次幂。
   - 执行的为收缩操作：ht[1].size 为 第一个大于等于 ht[0].used的 2的n次幂。

1. 将保存在字典ht[0]中的所有键值对rehash到ht[1]中。 rehash指重新计算hash值和索引值，然后将键值对放到ht[1]的相应位置。
2. 当ht[0]包含的所有键值对都迁移到了ht[1]之后，就将ht[0]释放，将ht[1]设置为ht[0]，之后在ht[1]处重新开一个空白哈希表，以便下次rehash使用。



#### 2.1.4 渐进式Rehash

当哈希表中有百万、千万级的数据需要rehash时，不能直接hash，否则会导致服务器很久没有响应。因此，为了避免rehash造成影响，redis会 渐近地进行rehash。

具体步骤：

- 为ht[1]分配好足够的空间
- 在字典中维护一个索引计数器变量rehashidx，将其设置为0，表示rehash开始。
- 每当字典处理一个正常操作时，都会顺带着执行将rehashidx指向ht[0]中的那条链表rehash到ht[1]中的rehash操作。当rehashidx处的rehash工作完成后，将rehashidx加一。
- 当ht[0]中的所有节点被搬到了ht[1]中后，程序将rehashidx重新设置为-1，并进行将ht[1]作为ht[0]，重新分配ht[1]等操作。



在渐进式rehash过程中，redis对字典的操作和平常有所不同。

- 进行插入操作时，将数据插入到ht[1]中，以此来避免插入到ht[0]中已经被rehash的部分。

- 在进行查找、删除、更新等操作时，会在两个哈希表上进行，一旦在ht[0]中找不到，就会在ht[1]中进行查找。



### 2.2 SkipList 跳表 (Zset的底层实现)

sikplist 是一种有序数据结构，他通过在节点中维护多个指向其他节点的指针来达到快速访问节点的目的。

他支持平均O(lgN)，最坏O(N)的查找，还可以通过顺序性操作来批量处理节点。

因为其实现简单，效率也较高，因此常被用来代替平衡树（红黑树？）。

Redis使用跳跃表来作为有序集合**键**的底层实现。



#### 2.2.1 数据结构

```c
typedef struct zskiplistNode {
    // 层
    struct zskiplistLevel {
        // 前进指针
        struct zskiplistNode* forward;
        // 跨度
        unsigned int span;
    } level[];
    // 后退指针
    struct zskiplistNode* backward;
    // 分值
    double score;
    // 成员对象
    robj* obj;
}
```

跳跃链表是在原来链表的基础上，结合了**二分查找**的一些思想进行了改造。

跳跃链表有很多层，最低一层的数据可以视为`未改造前的链表`，在这个单链表的基础上，跳跃链表通过随机函数，使得某些链表节点上有多个层`（skiplist在创建该节点时便用随机函数决定了这个节点有多少层了）`，**层数数值越大**的概率越小。用这些层来帮助跳跃链表来实现快速查找替换删除等。

![img](https://wx2.sinaimg.cn/mw690/006FwlnBly1g1hjfrlnt8j311m05ota4.jpg)

如图便为`为改造前的链表`

![img](https://wx2.sinaimg.cn/mw690/006FwlnBly1g1hjfrpbgnj311e0bo772.jpg)

该链表对应的skiplist如下，可见在某些节点上有多层。每次查询、插入、删除时，便通过这个多层的结构来实现跳跃。

#### 2.2.2 查找、插入与删除

**查找**

从最上一层节点开始查找。

如果当前节点的下一个节点的值小于目标值(`curr->next_nodes[i] < key`)，那么就将当前节点在该层向前一步。

如果发现当前节点的下一个节点的值大于目标值(`curr->next_nodes[i] > key`)（或到了最后，指向NULL），那么就进入低一层继续往后遍历。

如果发现当前节点的下一个节点值等于目标值，则返回下一个节点值，即为找到了目标。

**插入**

- 查找到小于 **将被插入的节点** 最大值，即前驱节点。
- 构建将被插入的节点，包括值的赋予 以及 层数的决定。假设此节点层数为k层(0到k-1)。
- 将该节点插入到0到k-1层中。

**删除**

- 在每一层上找到待删除节点的前一个节点
- 找到被删除的节点，将每一层中其pre指向其next，即删除该节点。
- 释放该节点的空间。



### 2.2.3 计算排名

`zskiplistLevel`中的跨度span用于计算某个节点的排名。

从开始到查找到该节点的路径中所有跨度的和便为其排名。



# 3.使用场景

- 计数器：使用String的自增自减运算来实现。
- 缓存：将热点数据放到内存中，设置缓存的最大使用量和淘汰策略来保证缓存的命中。
- 消息队列：使用List(双向链表)来写入和读取消息。
- 分布式锁？



# 4.Key过期机制

Redis可以为每个键设置过期时间，当键过期时，就会自动删除该键。

Redis的过期机制分为 **定时遍历策略** 和**惰性删除策略**

**定时遍历策略：**

redis将设有过期时间的key放入一个dict，然后定期（默认1s）遍历这个dict中key是否过期。

- 从dict中取出20个key (此dict为 将设有过期时间的key 放入的那个)。
- 删除这20个key中的过期的key
- 如果步骤2中的过期key超过1/4，就重复步骤1

因此redis可能会持续扫描过期字典（循环多次），直到过期字典中过期key变得稀疏，才会停止。因此可能会造成其他读写请求的卡顿。

因此，如果要再redis中加入大量设有过期时间的key，需要让过期时间在一个随机范围内，不能让很多key同时过期。



**惰性删除策略：**

在客户端访问这个key的时候，redis对key的过期时间进行检查，过期了就立即删除。



# 5.数据淘汰策略

可以设置Redis的最大内存使用量，当内存使用量超出时，就会自动实施淘汰策略。



### 5.1Redis缓存淘汰策略：

| 策略            | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
| volatile-lru    | 在**已设置过期时间的数据集中**挑选**最近最少使用**的数据淘汰。 |
| volatile-ttl    | 在**已设置过期时间的数据集中**挑选**将要过期**的数据淘汰。   |
| volatile-random | 在**已设置过期时间的数据集中**挑选**随机数据**淘汰。         |
| volatile-lfu    | 在**已设置过期时间的数据集中**挑选**访问频率最低**的数据淘汰。 |
| allkeys-lru     | 在**所有数据集**中挑选**最近最少使用**的数据淘汰。           |
| allkeys-random  | 在**所有数据集**中挑选**最近最少使用**的数据淘汰。           |
| allkeys-lfu     | 在**所有数据集**中挑选**访问频率最低**的数据淘汰             |
| noeviction      | **禁止驱逐数据**                                             |

### 5.2 LRU算法

LRU: Least Recently Used. 最近最少使用算法。

常用于缓存更新、页面置换等。

#### 5.2.1 LRU的常见实现：

- 链表实现：

  使用一个链表来实现LRU。

  - 插入数据时插入到链表头部。
  - 访问一个数据时，将节点重新移动到链表头部。
  - 当删除时，删除链表尾部的数据，其为最近最少使用的数据。

- 哈希表 + 双向链表实现：

  哈希表：用于检查是否在缓存中。

  双向链表：用于实现LRU算法（即找出最近最少使用）

  插入：

  - 先看哈希表中是否有，如果有则直接将原来的节点放到链表头。如果没有，则判断链表是否是满的。
    - 若为满的，移除链表尾部的元素，将这个元素插入到头部。
    - 若不满，则直接插入到头部。

  查找：

  - 直接用哈希表搜索key是否存在。若存在则直接返回该节点的位置，不存在返回NULL。



#### 5.2.2 Redis中的LRU

Redis为了节省内存使用，和通常的LRU算法实现不太一样，Redis使用了采样的方法来模拟一个近似LRU算法。

如果使用 双向链表 + 哈希表 来实现 LRU，当key数量比较多时，会比较消耗内存。Redis对每个key增加一个24bit的时钟，作为淘汰的依据，以此节省内存。



# 6.持久化:RDB和AOF 

- RDB持久化

  将某个时间点的所有数据都放到硬盘上。

  可以将快照复制到其他服务器从而创建具有相同数据的服务器副本。

  如果系统发生故障，将会丢失最后一次创建快照之后的数据。

  当数据量很大时，保存快照的时间会很长。

- AOF持久化

  将写命令添加到AOF(Append Only File)文件的末尾。

  AOF持久化可选的模式有三个：

  | 选项     | 同步频率               | 性能                                     | 崩溃丢失                 |
  | -------- | ---------------------- | ---------------------------------------- | ------------------------ |
  | always   | 每个写命令都           | 严重降低服务器性能                       | 基本不会丢失数据         |
  | everysec | 每秒同步一次           | 对服务器性能影响最小                     | 只会丢失1s的数据         |
  | no       | 操作系统来决定何时同步 | 相比everysec不会给服务器性能带来多大提升 | 相比everysec丢失数据更多 |

  AOF重写：随着服务器写请求增多，AOF文件会越来越大，Redis可以将AOF中冗余的写命令删除。



# 7.事务

### 7.1ACID特性

Redis中的事务只满足了**一致性与隔离性**，并没有满足**原子性和持久性**。

原子性：Redis中事务在执行中遇到错误时，不会回滚，而是继续执行后续的命令。但是**单个Redis命令的执行是原子性**的。

持久性：事务的持久性由Redis的持久化模式而决定（RDB或AOF）。



### 7.2 相关命令

- **MULTI + EXEC** Redis事务的基础命令

  首先调用MULTI，之后可以执行数条命令，在EXEC之后，将包裹其中的命令当作一个事务进行执行。

- **WATCH命令**

  在事务之前可以WATCH 某个key。

  在事务EXEC前，如果WATCH的值被其他客户端修改过了，那么Redis服务器就会放弃这个事务的执行。

- **DISCARD命令**

  将在命令队列中的所有命令都清除。实现上是将所有命令所在的存储空间都释放。



# 8.事件

Redis服务器是一个事件驱动的程序。



### 8.1 文件事件

Redis服务器通过套接字与Redis客户端和其他Redis服务器通信。将套接字视为文件。

文件事件便是对套接字操作的抽象。

Redis使用`I/O多路复用`来同时监听多个套接字，并将到达的事件分配给`文件事件分派器`，分派器会根据套接字产生的事件类型调用相应的处理程序。

![img](https://gitee.com/CyC2018/CS-Notes/raw/master/docs/pics/9ea86eb5-000a-4281-b948-7b567bd6f1d8.png)





### 8.2 时间事件

Redis还需要处理一些**定时事**件或**周期性事件**，他们统称为**时间事件**。

Redis将所有的时间事件都放在一个无序链表中。通过**遍历整个链表**来找出应该执行的时间事件，并根据不同事件类型调用相应的事件处理器。



### 8.3 Redis 运行流程

从事件处理的角度来看，Redis-Server的运行流程如下：

![img](https://gitee.com/CyC2018/CS-Notes/raw/master/docs/pics/c0a9fa91-da2e-4892-8c9f-80206a6f7047.png)

Redis服务器的主函数 伪代码如下：

```c
void main() {
    init_server();  // 初始化服务器
    while server_is_not_shutdown():  // Server还没有被停职
    	aeProcessEvents();  		 // 调用Redis自己实现的事件处理函数
    clean_server(); // 关闭服务器
}
```

aeProcessEvents(伪代码)如下：

```c
void aeProcessEvents() {
    // 找到到达事件离当前事件最接近的时间事件
    time_event = aeSearchNearestTime();
    // 计算最接近的时间事件距离到达还有多少毫秒
    remain_ms = time_event.when - unix_ts_now();
    // 如果时间事件已经该到达，那么remain_ms可能为负数，将其设置为0
    if remain_ms < 0:
    	remain_ms = 0;
    
    // 根据remain_ms创建timeval
    timeval = create_timeval_with_ms(remaind_ms);
    // 阻塞等待文件事件产生，如果超出了最大阻塞事件就会返回。
    // aeApiPoll底层调用的就是**I/O多路复用函数**！
    aeApiPoll(timeval);
    // 处理所有已经产生的文件事件
    processFileEvents();
    // 处理所有已经到达的时间事件
    processTimeEvents();
}
```



# 9. 主从服务器

可以使用 slaveof host port 命令让一个服务器成为另一个服务器的从服务器。

一个从服务器只能有一个主服务器。

主从服务器可以提升Redis的可用性，当一个节点挂掉时，写入操作可以由其他节点处理，避免了Redis重启服务后丢失数据的情景。



### 9.1连接过程

1. **主服务器**创建RDB快照文件，发送给从服务器。在发送期间使用缓冲区来记录执行的写命令。等快照发送完毕后，再开始向从服务器发送存储在缓冲区中的写命令。
2. 从服务器**丢弃所有旧数据**，载入主服务器发来的快照文件，之后从服务器开始接受主服务器发来的写命令。
3. 主服务器每执行一次写命令就向从服务器发送相同的写命令。



### 9.2 主从链

当从服务器数目不断增多时，主服务器可能无法很快地更新所有从服务器。

为了解决这个问题，可以使用中间层服务器来分担主服务器的工作。主服务器只需要将数据传输给中间层服务器即可。由中间层服务器将数据再复制给其他底层从服务器。



![img](https://gitee.com/CyC2018/CS-Notes/raw/master/docs/pics/395a9e83-b1a1-4a1d-b170-d081e7bb5bab.png)



### 9.3 Sentinel 哨兵

Sentinel可以监听集群心中的服务器，当主服务器下线时，可以自动在从服务器中选举出新的主服务器。

在Leader选举中，Sentinel底层使用了Raft算法。





# 10. 数据分片

将数据划分为多个部分，将数据存储到多台服务器中，以此来获得性能的提升。

分片策略：

- 按照数据某个值的范围分片。如id 0~1000分在第一片，1000~2000分在第二片等。这样需要维护一个映射范围表，维护代价较高。
- 哈希分片。使用哈希函数将键转换为一个数字，再对实例数量取余即得出应该存储在的实例位置。

分片方式：

- 客户端分片：Redis客户端使用一致性hash算法来决定应当分布到哪个节点。
- 代理分片：将客户端请求发送到代理上，由代理转发请求到正确的节点上。
- 服务器器集群分片。



Refs:

- Redis设计与实现第二版
- https://github.com/yuyilei/Daily-Notes/blob/master/md/redis-dict-rehash.md
- https://cyc2018.github.io/CS-Notes/#/notes/Redis
- https://www.jianshu.com/p/fcd18946994e
- Redis事务的ACID特性的讨论：https://www.cnblogs.com/chenpingzhao/p/5001894.html


