# 0 前言
参考文章：
使用redis组为缓存    https://www.cnblogs.com/leo-chen-2014/p/9739563.html
什么是redis雪崩、穿透和击穿    https://zhuanlan.zhihu.com/p/74880843



# 1 redis作用
常规业务系统的数据库访问中，读写操作的比例一般在7/3到9/1，也就是说读操作远多于写操作，因此高并发系统设计里，通过NoSQL技术将热点数据（短期内变动概率小的数据）放入内存以达到减轻DB压力，提升数据访问速度的目的，Redis和MongoDB是当下应用最广泛的NoSQL产品。

当然如果系统里的写操作居多，也没有必要使用缓存。

因此Redis主要用于解决**访问性能和并发能力**的问题。

除了纯数据缓存的作用之外，得益于其超高速的响应能力，Redis也常用于**提供分布式锁的解决方案**。

# 2 缓存雪崩
雪崩就是指缓存中大批量热点数据过期 或者 缓存系统本身宕机 后，数据库不得不直接接收大量查询请求，因为大部分数据在Redis层已经失效，请求渗透到数据库层，大批量请求犹如洪水一般涌入，引起数据库压力造成查询堵塞甚至宕机。

如果不采取措施，又尝试强行启动数据库，在热点数据没有被重新缓存的情况下，数据库立马
又被新的流量给打死了。循环往复，数据库始终处于无法稳定使用的状态。

## 2.1 解决方案

### 2.1.1 事前
redis 高可用，主从+哨兵，redis cluster，避免全盘崩溃，比如：

1. 将缓存失效时间分散开，比如每个key的过期时间是随机，防止同一时间大量数据过期现象发生，这样不会出现同一时间全部请求都落在数据库层，如果缓存数据库是分布式部署，将热点数据均匀分布在不同Redis和数据库中，有效分担压力，别一个人扛。
    
2. 简单粗暴，让Redis数据永不过期（如果业务准许，比如不用更新的名单类）。当然，如果业务数据准许的情况下可以，比如中奖名单用户，每期用户开奖后，名单不可能会变了，无需更新。

### 2.1.2 事中
本地 ehcache 缓存 + hystrix 限流&降级，避免 MySQL 被打死

### 2.1.3 事后
redis 持久化，一旦重启，自动从磁盘上加载数据，快速恢复缓存数据。

用户发送一个请求，系统 A 收到请求后，先查本地 ehcache 缓存，如果没查到再查 redis。如果 ehcache （ehcache介绍  https://www.jianshu.com/p/154c82073b07）和 redis 都没有，再查数据库，将数据库中的结果，写入 ehcache 和 redis 中。

限流组件，可以设置每秒的请求，有多少能通过组件，剩余的未通过的请求，怎么办？走降级！可以返回一些默认的值，或者友情提示，或者空白的值。

好处：
* 数据库绝对不会死，限流组件确保了每秒只有多少个请求能通过。
* 确保了整个系统是可用的状态。只要数据库不死，就是说，对用户来说，2/5 的请求都是可以被处理的。只要有 2/5 的请求可以被处理，就意味着你的系统没死，对用户来说，可能就是点击几次刷不出来页面，但是多点几次，就可以刷出来一次。

# 3 缓存穿透
对于系统A，假设一秒 5000 个请求，结果其中 4000 个请求是黑客发出的恶意攻击。黑客发出的那 4000 个攻击，缓存中查不到，每次你去数据库里查，实际上也是查不到。但是，这大量的请求直接穿透缓存，负载直接落到数据访问层。

举个栗子。数据库 id 是从 1 开始的，结果黑客发过来的请求 id 全部都是负数。这样的话，缓存中不会有，请求每次都“视缓存于无物”，直接查询数据库。这种恶意攻击场景的缓存穿透就会直接把数据库给打死。

## 3.1 解决方案
解决方式很简单，每次系统 A 从数据库中只要没查到，就写一个空值到缓存里去，比如 set -999 UNKNOWN。然后设置一个过期时间，这样的话，下次有相同的 key 来访问的时候，在缓存失效之前，都可以直接从缓存中取数据。


# 4 缓存击穿
缓存击穿，就是说某个 key 非常热点，访问非常频繁，处于集中式高并发访问的情况，当这个 key 在失效的瞬间，大量的请求就击穿了缓存，直接请求数据库，就像是在一道屏障上凿开了一个洞。

**雪崩就是大量的缓存击穿引发的后果，造成数据库持续的不可用。**

## 4.1 解决方案
解决方式也很简单，可以将热点数据设置为永远不过期；或者基于 redis or zookeeper 实现互斥锁（读 和 缓存构建 形成互斥，并发读不互斥），等待第一个请求构建完缓存之后，再释放锁，进而其它请求才能通过该 key 访问数据。

缓存构建和读取的互斥锁代码实现：
```java
public String get(key) {
    String value = redis.get(key);

    if (value == null) { //代表缓存值过期
    
        //设置互斥锁3min自动超时，
        //防止互斥锁释放（del key_mutex）操作失败，下次缓存过期时一直无法成功申请互斥锁而无限进行line 78-line79
        if (redis.setnx(key_mutex, 1, 3 * 60) == 1) {  //代表互斥锁设置成功
            value = db.get(key);
            redis.set(key, value, expire_secs);
            redis.del(key_mutex);
            return value;

        } else { //这个时候代表同时候的其他线程已经load db并回设到缓存了，这时候重试获取缓存值即可
            sleep(50);
            return get(key);  //重试
        }

    } else {
        return value;      
    }

}
```