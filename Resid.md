# Resid

## 1. Redis内存统计

> info memory

* used_memory：Redis分配器分配的内存总量

* used_memory_rss：Redis进程占操作系统的内存（除了分配器分配内存，还包括Redis进程占用内存，以及内存碎片）

* mem_fragmentation_ration：内存碎片率，该值是 used_memory_rss/used_memory，

  该值一般大于1，且该值越大，碎片率就越高，如果该值<1说明采用了 虚拟内存，需要进行扩容

* memory_allocator：Redis默认使用的内存分配器

## 2. Redis 内存划分

* 数据：这部分占用内存会存放在used_memory中
* 进程内存：这部分内存不是有jemalloc分配的，因此不会统计在used_memory中
* 缓冲内存：客户端缓冲，AOF 缓冲区，复制积压缓冲区

* 内存碎片

## 3. Redis数据存储细节

### 1. 概述

![image-20221013100057810](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221013100057810.png)

1. dictEntry：Redis是 k-v数据库，每一个键值对都对应一个dictEntry
2. key：字符串存放在SDS结构中
3. value：存放在redisObject中

### 2. jemalloc

jemolloc 作为Redis默认的内存分配器，将内存空间分为 小 大 巨大 三个范围，每个范围又划分了褫夺小的内存单位

### 3. redisObject

1. type：表示对象类型
2. encoding：表示对象编码
3. lru：对象被最后一次访问时间
4. refCount：对象被引用次数
5. ptr：指针

### 4. SDS结构

```c++
`struct` `sdshdr {`` ``int` `len;`` ``int` `free``;`` ``char`
`buf[];``};`
```

![image-20221013101711853](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221013101711853.png)

## 4. Redis对象类型与内部结构

字符串长度不能超过512M

### 1. 字符串

#### 1.内部编码

* int：8字节长整型
* embstr：<=44字节的字符串 (jemalloc只分配一次内存)
* raw：> 44 字节字符串 

#### 2. 编码转换

当 int数据不再是整数，或大小超过了long范围，自动转化为raw

emstr是只读的，只要修改过后，就会自动转化为raw

### 2. 列表

#### 1.内部编码

* 压缩列表
* 双端列表

#### 2.编码转换

* 列表长度 < 512 && 每个字符串对象< 64字节  压缩列表

### 3. 哈希

#### 1.内部编码

* 压缩列表

* 哈希表

  ![image-20221013102456116](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221013102456116.png)

#### 2.编码转换

* 哈希中元素数量小于512个；

* 哈希中所有键值对的键和值字符串长度都小于64字节。

### 4. 集合

#### 1.编码实现

* 整数集合

  ```
  typedef struct intset{
  //编码方式
  uint32_t encoding;
  //集合包含的元素数量
  uint32_t length;
  //保存元素的数组
  int8_t contents[];
  }intset;
  ```

  

* 哈希表 （value值全为 null）

#### 2.编码转换

* 集合中元素数量小于512个；

* 集合中所有元素都是整数值。

### 5.有序集合

#### 1.编码实现

* 压缩列表

  ![image-20221013103140099](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221013103140099.png)

* 跳表

  ![image-20221013102839896](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221013102839896.png)

#### 2. 内部转换

* 有序集合中数量 <128 且有序集合中每个值 <64字节 采用压缩列表

## 5. 内存优化

### 1.优化内存占用

### 2.关注内存碎片率

## 6.缓存淘汰策略

### 1. redis的配置

* maxmemory：默认值为0，没有指定最大缓存，如果有薪的数据添加，超过最大内存，则会使redis崩溃
* maxmemory oltile-lru：缓存淘汰策略

### 2. 缓存策略

* volatile-lru：从已设置过期时间的数据集（server.db[i].expires）中挑选最近最少使用的数据淘汰

* volatile-ttl：从已设置过期时间的数据集（server.db[i].expires）中挑选将要过期的数据淘汰

* volatile-random：从已设置过期时间的数据集（server.db[i].expires）中任意选择数据淘汰

* allkeys-lru：从数据集（server.db[i].dict）中挑选最近最少使用的数据淘汰

* allkeys-random：从数据集（server.db[i].dict）中任意选择数据淘汰

*  no-enviction（驱逐）：禁止驱逐数据

## 7.Redis的事务

### 1. 事务的命令

> multi：用于标记事务块的开始。Redis后会将后续命令放入队列中
>
> exec ：执行队列当中的命令，然后恢复正常连接的状态
>
> discard：清空队列中的命令，然后恢复正常连接状态
>
> watch：当某个事务需要按条件执行时，就要使用这个命令将指定的键设置为受监控的状态
>
> watch key [...]
>
> unwatch：清除所有先前为一个事务监控的键
>
> 

### 2.通过事务实现乐观锁

```java
public void watch() {
    try {
        String watchKeys = "watchKeys";
        //初始值 value=1
        jedis.set(watchKeys, 1);
        //监听key为watchKeys的值
        jedis.watch(watchkeys);
        //开启事务
        Transaction tx = jedis.multi();
        //watchKeys自增加一
        tx.incr(watchKeys);
        //执行事务，如果其他线程对watchKeys中的value进行修改，则该事务将不会执行
        //通过redis事务以及watch命令实现乐观锁
        List<Object> exec = tx.exec();
        if (exec == null) {
            System.out.println("事务未执行");
        } else {
            System.out.println("事务成功执行，watchKeys的value成功修改");
        }
        } catch (Exception e) {
         e.printStackTrace();
        } finally {
         jedis.close();
        }
}

// 通过乐观锁实现秒杀
public class Second {
    public static void main(String[] arg) {
        String redisKey = "second";
        ExecutorService executorService = Executors.newFixedThreadPool(20);
        try {
            Jedis jedis = new Jedis("127.0.0.1", 6378);
            // 初始值
            jedis.set(redisKey, "0");
            jedis.close();
        } catch (Exception e) {
       	 e.printStackTrace();
        }
        for (int i = 0; i < 1000; i++) {
            executorService.execute(() -> {
                Jedis jedis1 = new Jedis("127.0.0.1", 6378);
                try {
                    jedis1.watch(redisKey);
                    String redisValue = jedis1.get(redisKey);
                    int valInteger = Integer.valueOf(redisValue);
                    String userInfo = UUID.randomUUID().toString();
                    // 没有秒完
                    if (valInteger < 20) {
                        Transaction tx = jedis1.multi();
                        tx.incr(redisKey);
                        List list = tx.exec();
                        // 秒成功 失败返回空list而不是空
                        if (list != null && list.size() > 0) {
                            System.out.println("用户：" + userInfo + "，秒杀成
                            功！当前成功人数：" + (valInteger + 1));
                        }
                    // 版本变化，被别人抢了。
                        else {
                            System.out.println("用户：" + userInfo + "，秒杀失
                        败");
                        }
                    }
                    // 秒完了
                    else {
                   	 System.out.println("已经有20人秒杀成功，秒杀结束");
                    }
                } catch (Exception e) {
               	 e.printStackTrace();
                } finally {
                	jedis1.close();
                }
            });
        }
        executorService.shutdown();
    }
}
```

## 7. Redis的持久化

### 1. RDB方式(默认方式)

#### 1.触发时机

* 配置文件中配置的持久化规则
* 执行 save 或bsave命令
* 执行 flushall命令
* 执行主从复制的第一次 全量同步过程

#### 2.设置快照规则

```
save  多少秒内   数据改变了多少
save 900 1 ： 表示15分钟（900秒钟）内至少1个键被更改则进行快照。
save 300 10 ： 表示5分钟（300秒）内至少10个键被更改则进行快照。
save 60 10000 ：表示1分钟内至少10000个键被更改则进行快照。
save "" : 不使用RDB存储
```

#### 3. RDB持久化原理

 ![image-20221013105858100](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221013105858100.png)

注意事项：

* Redis在快照过程中不会修改 RDB文件，只有在快照操作结束后，才会将旧的文件替换为新的文件

优点：

* RDB可以最大化Redis进程，父进程fork出一个子进程，然后这个子进程会处理所有的保存工作，父进程无需进行任何磁盘IO操作

缺点：

* 一旦 redis异常退出，就会丢失最新的数据，而且当文件比较大时fork比较耗时，影响CPU性能

### 2. AOF方式

开启AOF后，没执行一条更改Redis中数据的命令，Redis就会将该命令写入到磁盘中的AOF文件

#### 1.开启AOF

```
# 将redis.conf 配置文件中的 appendonly 设置为yes
appendonly yes
# AOF文件的保存位置 通过dir参数设置
dir ./
# 设置文件名称
appendfilename appendonly.aof
```

#### 2. 同步磁盘的数据

Redis 每次更改数据时，会首先将命令写入硬盘缓存，再通过硬盘缓存机制刷新到文件

```
# 每次执行写入都会进行同步， 这个是最安全但是是效率比较低的方式
appendfsync always
# 每一秒执行(默认)
appendfsync everysec
# 不主动进行同步操作，由操作系统去执行，这个是最快但是最不安全的方式
appendfsync no
```

### 3. AOF重写

redis可以在AOF文件体积变得过大时，自动的在后台进行AOF重写，重写后的新的AOF文件 包含了恢复当前数据集所需的最小命令集合 

参数配置

```
# 表示当前aof文件大小超过上一次aof文件大小的百分之多少的时候会进行重写。如果之前没有重写
过，以启动时aof文件大小为准
auto-aof-rewrite-percentage 100
# 限制允许重写最小aof文件大小，也就是文件大小小于64mb的时候，不需要进行优化
auto-aof-rewrite-min-size 64mb
```

## 8. Redis 主从复制

分类：

* 全量同步（只有从机第一次连接上主机 才进行全量同步）

  分为三个阶段

  * 同步快照阶段：Master创建并发送快照到Slave，Slave载入并解析快照。Master同时将此阶段所产生的写命令存储到缓冲区
  * 同步写缓冲阶段：Master向Slave同步存储在缓冲区的写操作命令
  * 同步增量阶段：Master向Slave同步写操作命令

  ![image-20221013114513048](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221013114513048.png)

* 增量同步

  Master每执行一个命令都会向Slave发送相同的写命令

## 9. 哨兵模式

主从复制的缺点：没有办法对master进行动态选举，需要使用Sentinel机制完成动态选举

哨兵进程的作用：

* 监控
* 提醒
* 自动故障迁移

故障判定原理分析

*  每个 Sentinel （哨兵）进程以每秒钟一次的频率向整个集群中的 Master 主服务器， Slave 从服务器以及其他 Sentinel （哨兵）进程发送一个 PING 命令。

* 如果一个实例（ instance ）距离最后一次有效回复 PING 命令的时间超过 down-after-milliseconds 选项所指定的值， 则这个实例会被 Sentinel （哨兵）进程标记为主观下线( SDOWN ）。

* 如果一个 Master 主服务器被标记为主观下线（ SDOWN ），则正在监视这个 Master 主服务器的所有 Sentinel （哨兵）进程要以每秒一次的频率确认 Master 主服务器的确进入了主观下线状态。

* 当有足够数量的 Sentinel （哨兵）进程（大于等于配置文件指定的值）在指定的时间范围内确认 Master 主服务器进入了主观下线状态（ SDOWN ）， 则 Master 主服务器会被标记为客观下线（ ODOWN ）。

* 在一般情况下， 每个 Sentinel （哨兵）进程会以每 10 秒一次的频率向集群中的所有Master 主服务器、 Slave 从服务器发送 INFO 命令。

* 当 Master 主服务器被 Sentinel （哨兵）进程标记为客观下线（ ODOWN ）时， Sentinel （哨兵）进程向下线的 Master 主服务器的所有 Slave 从服务器发送 INFO 命令的频率会从 10秒一次改为每秒一次。

*  若没有足够数量的 Sentinel （哨兵）进程同意 Master 主服务器下线， Master 主服务器的客观下线状态就会被移除。若 Master 主服务器重新向 Sentinel （哨兵）进程发送 PING 命令返回有效回复， Master 主服务器的主观下线状态就会被移除。

自动故障迁移

* 它会将失效 Master 的其中一个 Slave 升级为新的 Master , 并让失效 Master 的其他 Slave 改为复制新的 Master ；

* 当客户端试图连接失效的 Master 时，集群也会向客户端返回新 Master 的地址，使得集群可以使用现在的 Master 替换失效 Master。

* Master 和 Slave 服务器切换后， Master 的 redis.conf 、 Slave 的 redis.conf 和sentinel.conf 的配置文件的内容都会发生相应的改变，即， Master 主服务器的 redis.conf配置文件中会多一行 slaveof 的配置， sentinel.conf 的监控目标会随之调换。

## 10. Redis集群

### 1.架构图

![image-20221013131109664](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221013131109664.png)

**架构细节**

* 所有的redis节点都互相连接，内部使用二进制协议优化传输速度和带宽

* 节点上的fail是通过集群中超过半数节点检测失效时生效

* 客户端与redis节点直连

* redis-cluster 把所有的物理节点映射到[0-16383]slot上

  > Redis 集群中内置了 16384个哈希槽，当需要在 Redis 集群中放置一个 key-value 时，redis
  >
  > 先对 key 使用 crc16 算法算出一个结果，然后把结果对 16384 求余数，这样每个 key 都会对应
  >
  > 一个编号在 0-16383 之间的哈希槽，redis 会根据节点数量大致均等的将哈希槽映射到不同的节点

**容错**

* 节点失效判断：集群当中所有master参数投票。超过半数以上的master基点与其中一个master节点通信超时，认为该master节点挂掉
* 集群失效判断
  * 某个主节点挂掉，且没有从节点
  * 超过半数以上的Master挂掉

## 11. Redis 分布式锁

原理：采用Redis的单线程特性 对共享资源进行串行化处理

### 11.1 实现方式

**获取锁**

方式1

```java
public boolean getLock(String lockKey,String requestId,int expireTime){
    // NX:保证互斥性
    // EX 设置单位时间
    String result = jedis.set(lockKey,requestId,"NX","EX");
    if("OK".equals(result)){
        return true;
    }
    return false;
}
```

方式2: 并发可能会产生问题

```java
public boolean getLock(String lockKey,String requestId,int expireTime){
    Long result = jedis.setnx(lockKey,requestId);
    if(reuslt ==1){
   		 // 设置失效时间
		jedis.expire(lockKey,expireTime);
		return true
    }
    return false;
}
```

**释放锁：**

方式1： 推荐

```java
public static boolean releaseLock(String lockKey,String requestId){
    String script = "if redis.call('get',KEYS[1] == ARGV[1] then return redis.call('del',KEYS[1]) else return 0 end)";
    Object result = jedis.eval(script,Collections.singletonList(localKey),Collections.singletonList(requestId));
    if(result.equals(1L)){
        return true;
    }
    return false;
}
```

方式2:

```
public static void releaseLock(String lockKey,String requestId) {
    if (requestId.equals(jedis.get(lockKey))) {
    	jedis.del(lockKey);
    }
}
```



## 12. Redis 常见的问题

### 1. 缓存穿透

查询缓存中永远不存在的一个值，因为这个值数据库没有

对查询结果为空的情况也进行缓存，缓存时间设置短一点

###  2.缓存雪崩

大量的缓存突然失效，大量的数据访问压力到数据库身上

* key的失效值 分开
* 设置二级缓存
* 高可用

### 3.缓存击穿

单个缓存的有效期到了，失效了，而且这个时间点有大量的请求进来

采用分布式锁控制访问的线程



### 4.数据写

强一致性很难，追求 最终一致性（延时双删）

保证数据的最终一致性（延时双删）

1. 先更新数据库同时删除缓存项，等读的时候再填充

2. 2秒后再删除一次缓存项
3. 设置缓存的过期时间



