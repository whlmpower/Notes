# Redis知识点

## 1. Redis的持久化（参考[1](http://www.cnblogs.com/zhoujinyi/archive/2013/05/26/3098508.html)、[2](http://www.redis.cn/topics/persistence.html)）

Redis有两种持久化方式：RDB（snapshotting方式）以及AOF（Append Only File）。

### 1.1 RDB方式

RDB方式是将当前内存中的数据写入到一个rdb文件中。可以通过设置保存时间或者调用`SAVE`或`BGSAVE`两个命令进行持久化。这个写入的过程如下：

1. redis调用fork,现在有了子进程和父进程。

2. 父进程继续处理client请求，子进程负责将内存内容写入到临时文件。由于os的写时复制机制（copy on write)父子进程会共享相同的物理页面，当父进程处理写请求时os会为父进程要修改的页面创建副本，而不是写共享的页面。所以子进程的地址空间内的数 据是fork时刻整个数据库的一个快照。

3. 当子进程将快照写入临时文件完毕后，用临时文件替换原来的快照文件，然后子进程退出。

使用RDB方式的方法是在Redis的配置文件中为`save`一项赋值，默认的值是`900 1 300 10 60 10000`。这一项的值两个为一组(m,n)，意思是每m秒有n个请求的时候对当前数据库进行保存。

缺点：这种方法可能会导致数据的丢失，因为其对数据的持久化不是实时的，会丢失掉宕机前几分钟的数据。而且，在Redis占用的内存空间较大的时候，创建子进程所需要的时间也会大大增加，占用更多内存，这个时候如果使用RDB进行备份，会导致Redis性能大大下降。

### 1.2 AOF方式

使用AOF方式每当 Redis 执行一个改变数据集的命令时（比如 SET）， 这个命令就会被追加到 AOF 文件的末尾。这样的话， 当 Redis 重新启时， 程序就可以通过重新执行 AOF 文件中的命令来达到重建数据集的目的。

如果想要打开AOF功能，要在配置文件中将`appendonly`设置为`yes`。

但是这样做就有可能是AOF文件越来越大，这个时候就需要对AOF文件进行重写，将其中的指令化简合并，从而减小文件的大小。用户可以通过BGREWRITEAOF命令来进行这一操作，同时可以通过设置`auto-aof-rewrite-percentage`和`auto-aof-rewrite-min-size`两个参数让Redis自动执行这个命令。

AOF重写方式的过程如下：

1. redis调用fork ，现在有父子两个进程。
2. 子进程根据内存中的数据库快照，往临时文件中写入重建数据库状态的命令。
3. 父进程继续处理client请求，除了把写命令写入到原来的aof文件中。同时把收到的写命令缓存起来。这样就能保证如果子进程重写失败的话并不会出问题。。
4. 当子进程把快照内容写入已命令方式写到临时文件中后，子进程发信号通知父进程。然后父进程把缓存的写命令也写入到临时文件。
5. 现在父进程可以使用临时文件替换老的aof文件，并重命名，后面收到的写命令也开始往新的aof文件中追加。

从上面看出，RDB和AOF操作都是顺序IO操作，性能都很高。而同时在通过RDB文件或者AOF日志进行数据库恢复的时候，也是顺序的读取数据加载到内存中。所以也不会造成磁盘的随机读。

到底选择什么呢？下面是来自官方的建议：

通常，如果你要想提供很高的数据保障性，那么建议你同时使用两种持久化方式。如果你可以接受灾难带来的几分钟的数据丢失，那么你可以仅使用RDB。

很多用户仅使用了AOF，但是我们建议，既然RDB可以时不时的给数据做个完整的快照，并且提供更快的重启，所以最好还是也使用RDB。

在数据恢复方面：

RDB的启动时间会更短，原因有两个：

一是RDB文件中每一条数据只有一条记录，不会像AOF日志那样可能有一条数据的多次操作记录。所以每条数据只需要写一次就行了。

另一个原因是RDB文件的存储格式和Redis数据在内存中的编码格式是一致的，不需要再进行数据编码工作，所以在CPU消耗上要远小于AOF日志的加载。

## 2. Redis速度快的原因

虽然Redis使用的是**单线程工作模型**，但是其操作仍然是非常快的，原因如下：

1. Redis中的数据操作都是内存操作；
2. 单线程工作，避免了上下文切换的过程；
3. 采用了非阻塞的IO多路复用技术。

## 3. Redis过期策略及内存淘汰机制

Redis对于过期键的删除，会采用**定期删除+惰性删除**的策略。

定期删除指的是Redis默认每100ms进行一次**随机抽样**检查，将检查到的**过期**的键删除掉。

> 注意，定期检查不会检查所有的键，因为如果检查所有的键会导致服务器性能降低。抽样检查也就会导致并不是所有过期的键都可以被及时删除。

惰性删除指的是Redis取到某个键之后，会检查以下这个键的过期时间，如果当前时间超过了过期时间，就将这个键删除掉。

但是当Redis内存空间不足的时候，就会启动内存淘汰机制。Redis配置文件中有`maxmemory-policy`一项，这项配置了Redis的内存淘汰算法，主要有以下几种：

1. noeviction：内存不足的时候如果再插入键会返回错误，很少用。
2. allkeys-lru：内存不足的时候，在所有键空间中，淘汰最近最少使用的key，推荐使用。
3. allkeys-random：内存不足的时候，在所有键空间中随机淘汰key，较少使用。
4. volatile-lru：内存不足的时候，在设置了过期时间的键空间中，淘汰最近最少使用的key，**这种情况一般是将Redis既当缓存又当持久化的时候才是用**。
5. volatile-random：同上，只不过是随即删除设置了过期时间的键。
6. volatile-ttl：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，有更早过期时间的key优先移除。