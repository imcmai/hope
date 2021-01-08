# springboot使用redisson集成哨兵模式的redis集群
## 配置抽离
```code
redisson.sentinelAddresses=redis://172.16.0.1:26379,redis://172.16.0.2:26379,redis://172.16.0.3:26379
redisson.masterName=mymater
redisson.password=
```
> redis哨兵默认使用端口为26379，非直连的6379
## 配置类
```java
@ConfigurationProperties(prefix = "redisson")
@Data
public class RedissonProperties {

    private int timeout = 3000;

    private int slaveConnectionPoolSize = 200;

    private int masterConnectionPoolSize = 200;

    private String[] sentinelAddresses;

    private String masterName;

    private String password;
}
```
## 注入redissonclient
```java
@Configuration
@ConditionalOnClass(Config.class)
@EnableConfigurationProperties(RedissonProperties.class)
public class RedissonAutoConfiguration {
    @Autowired
    private RedissonProperties redissonProperties;
    @Bean
    RedissonClient redissonSentinel() {
        Config config = new Config();
        SentinelServersConfig serverConfig = config.useSentinelServers().addSentinelAddress(redissonProperties.getSentinelAddresses())
                .setMasterName(redissonProperties.getMasterName())
                .setTimeout(redissonProperties.getTimeout())
                .setMasterConnectionPoolSize(redissonProperties.getMasterConnectionPoolSize())
                .setSlaveConnectionPoolSize(redissonProperties.getSlaveConnectionPoolSize());
        if(StringUtils.isNotBlank(redissonProperties.getPassword())) {
            serverConfig.setPassword(redissonProperties.getPassword());
        }
        return Redisson.create(config);
    }
```
## 测试类
```java
public class RedisLockTest extends BaseTest {
    @Autowired
    RedissonClient redissonClient;
    @Test
    public void testLock() throws InterruptedException {
        RLock rLock = redissonClient.getLock("test1");
        rLock.lock();
    }
}
```
查看key的过期时间
![TTLtest1](../_media/TTLtest1.png),可以看到key是有超时时间的，默认时间是30s，同时为了避免30s业务没有执行完，redisson的看门狗(watch dog)机制会启动一个定时任务，默认每隔十秒会去把key续期为30S,参考官方的解答
>> 大家都知道，如果负责储存这个分布式锁的Redisson节点宕机以后，而且这个锁正好处于锁住的状态时，这个锁会出现锁死的状态。为了避免这种情况的发生，Redisson内部提供了一个监控锁的看门狗，它的作用是在Redisson实例被关闭前，不断的延长锁的有效期。默认情况下，看门狗的检查锁的超时时间是30秒钟，也可以通过修改Config.lockWatchdogTimeout来另行指定。