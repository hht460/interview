# 缓存穿透

​	定义：key对应的数据在数据源并不存在，每次针对此key的请求从缓存获取不到，请求都会到数据源，从而可能压垮数据源。比如用一个不存在的用户id获取用户信息，不论缓存还是数据库都没有，若黑客利用此漏洞进行攻击可能压垮数据库。

缓存穿透解决方案：

1、布隆过滤器

​    将所有可能存在的数据哈希到一个足够大的bitmap中，一个一定不存在的数据会被 这个bitmap拦截掉，从而避免了对底层存储系统的查询压力。

2、设置空值

​    如果一个查询返回的数据为空（不管是数据不存在，还是系统故障），我们仍然把这个空结果进行缓存，但它的过期时间会很短，最长不超过五分钟。

# 缓存击穿

​    定义：key对应的数据存在，但在redis中过期，此时若有大量并发请求过来，这些请求发现缓存过期一般都会从后端DB加载数据并回设到缓存，这个时候大并发的请求可能会瞬间把后端DB压垮。（热点数据）

缓存击穿解决方案：使用互斥锁(mutex key)

```
（1）、在缓存失效的时候（判断拿出来的值为空），不是立即去load db，而是先使用缓存工具的某些带成功操作返回值的操作（比如Redis的SETNX或者Memcache的ADD）去set一个mutex key，当操作返回成功时，再进行load db的操作并回设缓存；
否则，就重试整个get缓存的方 法。
注释：
SETNX，是「SET if Not eXists」的缩写（当key不存在时，设置成功）;

（2）、伪代码实现：
public String get(key) {
      String value = redis.get(key);
      if (value == null) { //代表缓存值过期
          //设置3min的超时，防止del操作失败的时候，下次缓存过期一直不能load db
            if (redis.setnx(key_mutex, 1, 3 * 60) == 1) {  //代表设置成功
                value = db.get(key);
                redis.set(key, value, expire_secs);
                redis.del(key_mutex);
            } else {  //这个时候代表同时候的其他线程已经load db并回设到缓存了，这时候重试获取缓存值即可
              sleep(50);
              get(key);  //重试
            }
      } else {
          return value;      
      }
 }
```

# 缓存雪崩

定义：当缓存服务器重启或者大量缓存集中在某一个时间段失效，这样在失效的时候，也会给后端系统(比如DB)带来很大压力。

缓存雪崩解决方案：

```
与缓存击穿的区别在于这里针对很多key缓存，前者则是某一个key。

（1）、增加互斥锁，控制数据库请求，重建缓存；
//伪代码
public object GetProductListNew() {
    int cacheTime = 30;
    String cacheKey = "product_list";
    String lockKey = cacheKey;

    String cacheValue = CacheHelper.get(cacheKey);
    if (cacheValue != null) {
        return cacheValue;
    } else {
        synchronized(lockKey) {
            cacheValue = CacheHelper.get(cacheKey);
            if (cacheValue != null) {
                return cacheValue;
            } else {
              //这里一般是sql查询数据
                cacheValue = GetProductListFromDB(); 
                CacheHelper.Add(cacheKey, cacheValue, cacheTime);
            }
        }
        return cacheValue;
    }
}

（2）、为key设置不同的缓存失效时间；
```

# Redis几个过期策略

```
redis 设置key过期时间：
expire key time(以秒为单位) 这是最常用的方式
setex(String key, int seconds, String value) 字符串独有的方式
```

## 三种过期策略

### 1、定时删除

​	在设置key的过期时间的同时，为该key创建一个定时器，让定时器在key的过期时间来临时，对key进行删除；

优点：保证内存被尽快释放；

缺点：
1、若过期key很多，删除这些key会占用很多的CPU时间，在CPU时间紧张的情况下，CPU不能把所有的时间用来做要紧的事儿，还需要去花时间删除这些key
2、定时器的创建耗时，若为每一个设置过期时间的key创建一个定时器（将会有大量的定时器产生），性能影响严重

### 2、懒汉式式删除

​	key过期的时候不删除，每次通过key获取值的时候去检查是否过期，若过期，则删除，返回null
注：对于懒汉式删除而言，并不是只有获取key的时候才会检查key是否过期，在某些设置key的方法上也会检查（eg.setnx key2 value2）

优点：
	删除操作只发生在通过key取值的时候发生，而且只删除当前key，所以对CPU时间的占用是比较少的，而且此时的删除是已经到了非做不可的地步；

缺点：
	若大量的key在超出超时时间后，很久一段时间内，都没有被获取过，那么可能发生内存泄露（无用的垃圾占用了大量的内存）；

### 3、定期删除

每隔一段时间执行一次删除过期key操作

优点：
	通过限制删除操作的时长和频率，来减少删除操作对CPU时间的占用；

缺点：
	在内存友好方面，不如"定时删除"（会造成一定的内存占用）
	在CPU时间友好方面，不如"懒汉式删除"（（会定期的去进行比较和删除操作）	

难点：
	合理设置删除操作的执行时长（每次删除执行多长时间）和执行频率（每隔多长时间做一次删除）（这个要根据服务器运行情况来定了），每次执行时间太长，或者执行频率太高对cpu都是一种压力；
	每次进行定期删除操作执行之后，需要记录遍历循环到了哪个标志位，以便下一次定期时间来时，从上次位置开始进行循环遍历；

### 4、Redis采用的过期策略

懒汉式删除+定期删除

懒汉式删除流程：
	1、在进行get或setnx等操作时，先检查key是否过期；
	2、若过期，删除key，然后执行相应操作；
	3、若没过期，直接执行相应操作；

定期删除流程：
	1、遍历每个数据库（就是redis.conf中配置的"database"数量，默认为16）
	2、检查当前库中的指定个数个key（默认是每个库检查20个key，注意相当于该循环执行20次，循环体是下边的描述）
	3、如果当前库中没有一个key设置了过期时间，直接执行下一个库的遍历
	4、随机获取一个设置了过期时间的key，检查该key是否过期，如果过期，删除key
	5、判断定期删除操作是否已经达到指定时长，若已经达到，直接退出定期删除。
	6、对于定期删除，在程序中有一个全局变量currentdb来记录下一个将要遍历的库，假设有16个库，我们这一次定期删除遍历了10个，那此时的currentdb就是11，下一次定期删除就从第11个库开始遍历，假设currentdb等于15了，那么之后遍历就再从0号库开始（此时currentdb==0）

# redis分布式锁

## Redis服务端单机部署（单节点）

使用第三方开源组件jedis实现Redis客户端;

如果采用单机部署模式，会存在单点问题，只要redis故障了。加锁就不行了

### 获取锁

```
private static final String LOCK_SUCCESS = "OK";
private static final String SET_IF_NOT_EXIST = "NX";
private static final String SET_WITH_EXPIRE_TIME = "PX";

/**
 * 尝试获取分布式锁
 * @param jedis Redis客户端
 * @param lockKey 锁
 * @param requestId 请求标识
 * @param expireTime 超期时间
 * @return 是否获取成功
 */
public static boolean tryGetDistributedLock(Jedis jedis, String lockKey, String requestId, int expireTime) {
    String result = jedis.set(lockKey, requestId, SET_IF_NOT_EXIST, SET_WITH_EXPIRE_TIME, expireTime);
    if (LOCK_SUCCESS.equals(result)) {
        return true;
    }
    return false;
}
// redis原生命令
set lock 123456 nx px 10000
```

-- set方法解释：
jedis.set(String key, String value, String nxxx, String expx, int time)，这个set()方法一共有五个形参；
第一个为key，我们使用key来当锁，因为key是唯一的。
第二个为value，我们传的是requestId，很多童鞋可能不明白，有key作为锁不就够了吗，为什么还要用到value？原因就是我们在上面讲到可靠性时，分布式锁要满足第四个条件解铃还须系铃人，通过给value赋值为requestId，我们就知道这把锁是哪个请求加的了，在解锁的时候就可以有依据。requestId可以使用UUID.randomUUID().toString()方法生成。
第三个为nxxx，这个参数我们填的是NX，意思是SET IF NOT EXIST，即当key不存在时，我们进行set操作；若key已经存在，则不做任何操作；
第四个为expx，这个参数我们传的是PX，意思是我们要给这个key加一个过期的设置，具体时间由第五个参数决定。
第五个为time，与第四个参数相呼应，代表key的过期时间

### 释放锁

    private static final Long RELEASE_SUCCESS = 1L;
    
    /**
     * 释放分布式锁
     * @param jedis Redis客户端
     * @param lockKey 锁
     * @param requestId 请求标识
     * @return 是否释放成功
     */
    public static boolean releaseDistributedLock(Jedis jedis, String lockKey, String requestId) {
        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
        Object result = jedis.eval(script, Collections.singletonList(lockKey), Collections.singletonList(requestId));
        if (RELEASE_SUCCESS.equals(result)) {
            return true;
        }
        return false;
    }

## Redis cluster模式（集群模式）

RedLock的算法；
假设redis的部署模式是redis cluster，总共有5个master节点，通过以下步骤获取一把锁：
（1）、获取当前时间戳，单位是毫秒
（2）、轮流尝试在每个master节点上创建锁，过期时间设置较短，一般就几十毫秒
（3）、尝试在大多数节点上建立一个锁，比如5个节点就要求是3个节点（n / 2 +1）
（4）、客户端计算建立好锁的时间，如果建立锁的时间小于超时时间，就算建立成功了
（5）、要是锁建立失败了，那么就依次删除这个锁
（6）、只要别人建立了一把分布式锁，你就得不断轮询去尝试获取锁

## master-slave + sentinel选举模式（主从模式）

​	采用master-slave模式，加锁的时候只对一个节点加锁，即便通过sentinel做了高可用，但是如果master节点故障了，发生主从切换，此时就会有可能出现锁丢失的问题

## Redisson框架 实现分布式锁

***适用于redis三种部署模式***

```
Config config = new Config();
config.useClusterServers()
.addNodeAddress("redis://192.168.31.101:7001")
.addNodeAddress("redis://192.168.31.101:7002")
.addNodeAddress("redis://192.168.31.101:7003")
.addNodeAddress("redis://192.168.31.102:7001")
.addNodeAddress("redis://192.168.31.102:7002")
.addNodeAddress("redis://192.168.31.102:7003");

RedissonClient redisson = Redisson.create(config);

RLock lock = redisson.getLock("anyLock");
lock.lock();
lock.unlock();

我们只需要通过它的api中的lock和unlock即可完成分布式锁，他帮我们考虑了很多细节：

redisson所有指令都通过lua脚本执行，redis支持lua脚本原子性执行
redisson设置一个key的默认过期时间为30s,如果某个客户端持有一个锁超过了30s怎么办？
redisson中有一个watchdog的概念，翻译过来就是看门狗，它会在你获取锁之后，每隔10秒帮你把key的超时时间设为30s
这样的话，就算一直持有锁也不会出现key过期了（当前线程丢失锁），其他线程获取到锁的问题了。
redisson的“看门狗”逻辑保证了没有死锁发生。
(如果机器宕机了，看门狗也就没了。此时就不会延长key的过期时间，到了30s之后就会自动过期了，其他线程可以获取到锁)
```



https://zhuanlan.zhihu.com/p/366176522
