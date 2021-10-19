# 1.基于zk的分布式锁
## 1.1 基于curator框架实现
curator一个比较完善的ZooKeeper客户端框架，封装了一系列高级API简化ZooKeeper的操作，解决了：

  封装ZooKeeper client与ZooKeeper server之间的连接处理；
  
  提供了一套Fluent风格的操作API；
  
  提供ZooKeeper各种应用场景(recipe， 比如：分布式锁服务、集群领导选举、共享计数器、缓存机制、分布式队列等)的抽象封装。
  
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

###缺点

1. 加锁会频繁地“写”zookeeper，增加zookeeper的压力；

2. 写zookeeper的时候会在集群进行同步，节点数越多，同步越慢，获取锁的过程越慢；

3. 需要另外依赖zookeeper，而大部分服务是不会使用zookeeper的，增加了系统的复杂性；

4. 相对于redis分布式锁，性能要稍微略差一些；
 
