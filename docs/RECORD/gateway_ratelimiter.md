## 限流场景
1. 不健康的压力测试，或者脚本编写错误，导致网关或下游服务被占用大量资源，影响正常请求
2. 帮助下游服务，提供预期的流量，非预期流量将会被拒绝/阻塞
## spring cloud gateway是如何实现限流过滤器的
源码在RequestRateLimiterGatewayFilterFactory，由RequestRateLimiterGatewayFilterFactory的内置Config类来初始化限流过滤器,我们来看几个关键配置
```code
	public static class Config implements HasRouteId {
        // 限流的目标，可以自定义实现
		private KeyResolver keyResolver;

        // 限流的实现，默认是基于redis的令牌桶
		private RateLimiter rateLimiter;

        // 被拒绝的请求返回的状态码，默认429
		private HttpStatus statusCode = HttpStatus.
        TOO_MANY_REQUESTS;

        // 当限流目标为空，是否拒绝请求，默认为true
        private Boolean denyEmptyKey;

        // 限流目标为空，拒绝请求时返回的状态码，默认是403
		private String emptyKeyStatus;

		private String routeId;
        ....省略get/set
	}

```
### 内置的令牌桶算法
源码在RedisRateLimiter类中，RedisRateLimiter的Config类负责配置令牌生成和消费的规则
```code
	@Validated
	public static class Config {

        // 每秒生成的令牌数
		@Min(1)
		private int replenishRate;

        // 令牌桶最大容量
		@Min(0)
		private int burstCapacity = 1;

        //每次请求消耗的令牌
		@Min(1)
		private int requestedTokens = 1;
```
1. 采用延迟计算方式，每次扣除令牌之前推算桶中的令牌数
2. 初始化桶是满的

**为什么要使用redis?**

1. 要保证检查和扣除令牌动作的原子性，使限流更精准
2. 天然的分布式限流，网关在多节点的情况下共享限流数据
### 如何实践中使用
引入redis的pom文件
```code
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
        </dependency>
```
配置redis
```code
spring.redis.database = 
spring.redis.host = 
spring.redis.port = 
spring.redis.password=
```
配置过滤器
```code
        filters:
        - name: RequestRateLimiter
            args:
                deny-empty-key: false
                redis-rate-limiter.replenishRate: 1
                redis-rate-limiter.burstCapacity: 1
                redis-rate-limiter.requestedTokens: 1
                status-code: TOO_MANY_REQUESTS
                key-resolver: "#{@headerKeyResolver}"
```
#### 如何限制每分钟一次请求?
修改每次请求消耗的令牌以及桶最大的令牌，可以灵活运用
```code
        filters:
        - name: RequestRateLimiter
            args:
                deny-empty-key: false
                redis-rate-limiter.replenishRate: 1
                redis-rate-limiter.burstCapacity: 60
                redis-rate-limiter.requestedTokens: 60
                status-code: TOO_MANY_REQUESTS
                key-resolver: "#{@headerKeyResolver}"
```
### 自定义限流对象实现
```code
    @Bean(name = "headerKeyResolver")
    public KeyResolver headerKeyResolver() {
        KeyResolver keyResolver = exchange -> Mono.just( exchange.getRequest().getHeaders().getFirst(HEADER_KEY_ID_NAME));
        return keyResolver;
    }
```