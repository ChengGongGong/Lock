# 1.基于zk的分布式锁
## 1.1 基于curator框架实现
### 1.Curator
1. curator一个比较完善的ZooKeeper客户端框架，封装了一系列高级API简化ZooKeeper的操作，解决了：

    封装ZooKeeper client与ZooKeeper server之间的连接处理；
  
    提供了一套Fluent风格的操作API；
  
    提供ZooKeeper各种应用场景(recipe， 比如：分布式锁服务、集群领导选举、共享计数器、缓存机制、分布式队列等)的抽象封装。
  
2. 提供了异步接口。引入了BackgroundCallback 这个回调接口以及 CuratorListener 这个监听器，用于处理 Background 调用之后服务端返回的结果信息。
    所有操作需要加上inBackground()。
    
3. 连接状态监听，提供了监听连接状态的监听器——ConnectionStateListener，主要是处理 Curator 客户端和 ZooKeeper 服务器间连接的异常情况。

         短暂断开连接：zk会检测到该连接已经断开，但是服务端维护的客户端Session尚未过期，当重新连接后，由于Session没有过期能够保证连接恢复后保持正常服务。
         长时间断开连接：session已过期，与先前session相关的watcher和临时节点都会丢失，当重新连接时，会获取到 Session 过期的相关异常，Curator 会销毁老 Session，并且创建一个新的 Session。
                        由于老 Session 关联的数据不存在了，监听到 LOST 事件时，就可以依靠本地存储的数据恢复 Session 了。
         ZooKeeper 通过 sessionID 唯一标识 Session，将 Session 信息存放到硬盘中，通过置客户端会话的超时时间（sessionTimeout），即使节点重启，之前未过期的 Session 仍然会存在。

4. Watcher-监听机制，可以监听某个节点上发生的特定事件，例如监听节点数据变更、节点删除、子节点变更等事件。
    当相应事件发生时，ZooKeeper 会产生一个 Watcher 事件，并且发送到客户端。
    实现org.apache.curator.framework.api.CuratorWatcher接口，并重写其process方法。
    checkExists()、getData()和getChildren()添加usingWatcher方法
    
5. Cache- ZooKeeper 服务端事件的监听

        直接通过注册 Watcher 进行事件，需要反复注册Watcher，使用cache能够自动处理反复注册监听，看作是一个本地缓存视图和远程zk视图的对比过程，包括三大类：
        NodeCache：对一个节点进行监听，监听事件包括指定节点的增删改操作，还能监听指定节点是否存在；
        PathChildrenCache：对指定节点的一级子节点进行监听，监听事件包括子节点的增删改操作，但是不对该节点的操作监听。
        TreeCache：综合 NodeCache 和 PathChildrenCache 的功能，还可以设置监听的深度。参考：         org.apache.dubbo.remoting.zookeeper.curator.CuratorZookeeperClient#addTargetDataListener
### 2.curator-recipes
      Queues：提供了多种的分布式队列解决方法，比如：权重队列、延迟队列等。在生产环境中，很少将 ZooKeeper 用作分布式队列，只适合在压力非常小的情况下，才使用该解决方案，建议适度使用
      Counters：全局计数器是分布式系统中很常用的工具，curator-recipes 提供了 SharedCount、DistributedAtomicLong 等组件，帮助开发人员实现分布式计数器功能。
      Locks：(java.util.concurrent.locks)，在微服务架构中，分布式锁也是一项非常基础的服务组件，curator-recipes 提供了多种基于 ZooKeeper 实现的分布式锁，满足日常工作中对分布式锁的需求。
      Barries：curator-recipes 提供的分布式栅栏可以实现多个服务之间协同工作，具体实现有 DistributedBarrier 和 DistributedDoubleBarrier。
      Elections：实现的主要功能是在多个参与者中选举出 Leader，然后由 Leader 节点作为操作调度、任务监控或是队列消费的执行者。curator-recipes 给出的实现是 LeaderLatch。

 ### 引入相关pom文件
 
    <dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>4.0.0</version> 
    </dependency>

    <dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-framework</artifactId>
    <version>4.0.0</version>
   </dependency>
 
 ### 示例如下
 
 public class ZkCuratorLocker {

        public static final String connAddr = "xxx";
        public static final int timeout = 6000;
        public static final String LOCKER_ROOT = "/locker";

        private CuratorFramework cf;

        /**
         * connectTimeoutMS,连接超时，默认15s
         * sessionTimeoutMS，session超时，默认60s
         * retryPolicy，重连策略，四种:
         *  1.RetryUntilElapsed(int maxElapsedTimeMs, int sleepMsBetweenRetries),
         *      以sleepMsBetweenRetries的间隔重连,直到超过maxElapsedTimeMs的时间设置；
         *  2.RetryNTimes(int n, int sleepMsBetweenRetries)
         *      指定重连次数
         *  3.RetryOneTime(int sleepMsBetweenRetry)
         *      重连一次
         *  4.ExponentialBackoffRetry(int baseSleepTimeMs, int maxRetries)
         *    ExponentialBackoffRetry(int baseSleepTimeMs, int maxRetries, int maxSleepMs)
         *      时间间隔 = baseSleepTimeMs * Math.max(1, random.nextInt(1 << (retryCount + 1)))，
         *      时间间隔以指数形式增长
         **/
        @PostConstruct
        public void init() {
            this.cf = CuratorFrameworkFactory.builder()
                                             .connectString(connAddr)
                                             .sessionTimeoutMs(timeout)
                                             .retryPolicy(new ExponentialBackoffRetry(1000, 3))
                                             .build();

            cf.start();
        }

        /**
         *  zk中有四种节点类型：持久、持久有序、临时、临时有序
         *  zk的分布式锁基于临时有序节点+监听机制实现的，具体逻辑：
         *      1、加锁时，在锁路径下创建临时有序节点，使用UUID生成一个节点名称；
         *      2、获取该路径下的所有节点并对名称排序，判断新建的节点索引是否为0(第一个节点)；
         *      3、是，则返回节点路径，并获取到锁；不是，则监听前一个节点，并阻塞当前线程
         *      4、当监听到前一个节点的删除事件时，唤醒当前节点的线程，并再次检查自己是不是第一个节点
         * @param key
         */
        public void lock(String key) {
            String path = LOCKER_ROOT + "/" + key;
            InterProcessLock lock = new InterProcessMutex(cf, path);
            try {

                lock.acquire();
                //具体业务逻辑
            } catch (Exception e) {
                log.error("get lock error", e);

            } finally {
                try {
                    lock.release();
                } catch (Exception e) {
                    log.error("release lock error", e);
                }
            }
        }
    }
    
 ## 优缺点
 ### 优点
 
 1. zookeeper本身可以集群部署，相对于mysql的单点更可靠；
 
 2. 客户端断线能够自动释放锁，非常安全；
 
 3. 有现有的curator可以使用，且实现方式是可重入的，对现有代码改造成本小；
 
 4. 使用监听机制，减少线程上下文切换的次数；

### 缺点

1. 加锁会频繁地“写”zookeeper，增加zookeeper的压力；

2. 写zookeeper的时候会在集群进行同步，节点数越多，同步越慢，获取锁的过程越慢；

3. 需要另外依赖zookeeper，而大部分服务是不会使用zookeeper的，增加了系统的复杂性；

4. 相对于redis分布式锁，性能要稍微略差一些；
 
# 2.基于redis的分布式锁

## 2.1 基于redis单实例

@Slf4j

public class RedisLock {

    private static final String LOCK_SUCCESS="ok";

    private static final String UNLOCK_SUCCESS="1";

    private static final int DEFAULT_EXPIRE = 5;

    private static final String LOCK_PRE="lock:%s";

    private static final String UNLOCK_LUA=  "if redis.call('get',KEYS[1]) == ARGV[1]) then" +
                                             "return redis.call('del',KEYS[1])" +
                                             "else" +
                                             "return 0" +
                                             "end";

    @Autowired
    private JedisCluster jedisCluster;

    public boolean lock(String taskName,String value,int expireTime){
        if(expireTime<0){
            expireTime=DEFAULT_EXPIRE;
        }
        String lockKey=String.format(LOCK_PRE,taskName);
        //使用set nx ex给当前线程设置一个唯一的随机数
        try{
            if(jedisCluster.set(lockKey, value, SetParams.setParams().nx().ex(expireTime)).equals(LOCK_SUCCESS)){
                return true;
            }
        }catch (Exception e) {
          log.error("redis lock failed,task:{}",taskName,e);
        }

        return false;
    }

    public boolean unLock(String taskName,String value){
        String lockKey=String.format(LOCK_PRE,taskName);
        try {
            //使用lua脚本原子性的比较是不是当前线程加的锁和释放锁
            String result=(String)jedisCluster.eval(UNLOCK_LUA, Collections.singletonList(lockKey),Collections.singletonList(value));
            if(UNLOCK_SUCCESS.equals(result)){
                return true;
            }
        }catch (Exception e) {
            log.error("redis unLock failed,task:{}",taskName);
        }

        return false;
    }
}

//存在的问题：

1. redis集群模式下，会存在锁丢失的情况，当主节点挂掉时，主节点还没来得及同步到从节点，从节点升级为主节点，导致锁丢失；

2. 锁超时自动续期问题，给锁设置的过期时间太短，业务还没执行完成，锁就过期了

## 2.2 基于Redisson实现分布式锁-RedissonLock

1. 加锁机制，对于redis集群，通过hash算法选择一个节点，并执行lua脚本加锁，默认过期时间30s；
2. 锁互斥机制，不同的客户端执行相同的lua脚本，发现锁key的hash数据结构中field不是自己的线程id，则返回锁的剩余生存时间，并循环等待，不断尝试加锁。
3. watch dog自动续期机制，某个线程加锁成功，就会启动一个看门狗的后台线程，每隔10s检查一次，如果该线程仍持有锁，就不断的延长锁key的生存时间
4. 可重入机制，相同的客户端执行相同的lua脚本，发现锁key的hash数据结构中的filed是自己的线程id，则将value值加1
5. 锁释放机制，执行lua脚本，每次对锁key的hash数据结构中filed值减1，发现该vaule值为0，则执行删除锁key命令
 ### 缺点
 当redis主节点宕机时，可能会导致多个客户端同时完成加锁

## 2.3 基于Redisson实现分布式锁-RedissonRedLock

在分布式环境中,N个主节点，当且仅当大多数(N/2+1)的redis节点都获取到锁，且使用的时间小于锁失效的时间，锁才算获取成功；

如果获取锁失败，客户端应该在所有的Redis实例上进行解锁(即使某些redis实例没有加锁成功)
## 示例

 public static void main(String[] args) {
 
        Config config=new Config();
        //详细配置属性,详见:https://github.com/redisson/redisson/wiki/%E7%9B%AE%E5%BD%95
        config.useClusterServers()
              /**
               * 连接超时ms,
               * 最好不要低于500ms,线上出现多起因为连接超时过短，造成大量建连引发Redis雪崩
               */
              .setConnectTimeout(10000)
              /**
               *  redis 命令相应超时ms，合理配置
               */
              .setTimeout(2000)
              /**
               * master连接最小空闲数,
               * 保持masterConnectionMinimumIdleSize=masterConnectionMinimumIdleSize；
               * 建议根据需要设置( qps/主节点数/ (1000/每条命令的预计耗时ms)*(3/2)),且不超过业务线程数量;
               * 原则上不要超过50;例如： 主节点数：10,命令耗时:0.33ms, 10个连接的qps为: 10 * (1000ms/0.33ms) * 10 = 30w
               */
              .setMasterConnectionMinimumIdleSize(10)
              /**
               * master连接池大小,
               * 保持masterConnectionMinimumIdleSize=masterConnectionMinimumIdleSize
               */
              .setMasterConnectionPoolSize(10)
              // 添加多对主从作为种子节点
              .addNodeAddress("redis://xxx:9736")
              .addNodeAddress("redis://xxx:9736");

              RedissonClient redisson= Redisson.create(config);
              RLock lock1=redisson.getLock("lock1");
              RLock lock2=redisson.getLock("lock2");
              RLock lock3=redisson.getLock("lock3");

              RedissonRedLock lock=new RedissonRedLock(lock1,lock2,lock3);

              lock.lock();
              try{
                  //具体业务逻辑
              }finally {
                  lock.unlock();
              }
              
    }
    
## 缺点
 当短时间内有大量的加锁解锁操作，会导致网络流量限制和redis的cpu过载，主要是因为redis在锁中采用的发布订阅机制进行锁的监控，消息被分发到集群中的所有节点中去。
 
 redis中新增了Spin Lock(自旋锁)，采用指数回调的策略自旋获取锁而不是通过发布订阅机制
