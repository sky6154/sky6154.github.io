---
title:  "Spring에서 redis와 connection pool 및 알아두면 좋을 점"
date:   2023-10-06 14:33:21 +0900
categories:
  - spring
tags:
  - redis
excerpt: "What you need to know when using redis connection pool in spring"
toc: true
toc_sticky: true
---
## 개요
redis를 사용하면서 코드들을 보다가 문득 redis는 왜 connection pool을 사용하지 않는건지에 대한 의문점이 생겼다.  
(connection을 맺을 필요 없이 pooling 하면 보통 효율적인거 아닌가..?)  

관련해서 내용을 찾아보았고 역시 다 이유가 있구나.. 싶었다.  
찾아보았던 내용을 정리한다.  
해당 문서는 spring boot 2 기준으로 작성되었으나, 3버전에도 크게 다르지는 않을까 싶다.  

## 확인
Spring boot 프로젝트에서 redis를 사용하려면 다음과 같이 할 수 있다.

1. driver 구현체를 직접 library에 추가해서 사용한다.
2. spring-boot-starter-redis 를 사용해서 좀 더 간편하게 사용한다.

2번을 사용하고 있었는데, `build.gradle`에 다음과 같은 의존성을 사용하면 바로 사용할 수 있다.  

```yaml
//redis
implementation 'org.springframework.boot:spring-boot-starter-data-redis'
```

위와 같이 하고 application.yml에 다음 설정을 적용하면 바로 redis를 사용할 수 있게 된다.  
```yaml
spring:
  redis:
    database: 2
    host: localhost
    port: 6379
    lettuce:
      pool:
        max-active: 10
        max-idle: 10
        min-idle: 2
```

redis에 접근하기 위한 드라이버는 보통 `jedis`와 `lettuce`를 사용하는데, `jedis`의 경우 많은 사람들이 오래전부터 사용해왔지만, 어느순간 업데이트도 더뎌지고 속도면에서도 `lettuce`가 더 빠르다 라는 글들과 실제 테스트 결과들이 공유되기 시작했다.    
또한 spring boot에서도 어느 기점으로 default driver를 `lettuce`로 변경하여서 나도 몇번 테스트 해보고 `lettuce`를 사용중이다.

위 설정에서 `spring.redis` 부분은 redis의 `server` 설정이고, 아래 `lettuce` 하위 부분은 redis `client` 설정 부분이다.  

위 코드를 넣고 구동시켜보면 redis 호출도 잘 되고 connection pool을 사용해서 사용하는 것처럼 느껴진다.  
하지만 위 설정만으로는 redis connection pool을 사용할 수 없다.   

[spring 문서](https://docs.spring.io/spring-data/redis/docs/3.1.9/reference/html/#redis:connectors:connection) 를 참고하면 lettuce나 jedis 모두 connection pooling을 사용하려면 `common-pools2` 를 사용해야 한다고 되어 있다.  

![spring_doc]({{ "/assets/images/spring-and-redis-connection-pool/spring_doc.png" | relative_url }})

`spring-boot-starter-data-redis`를 추가해도 자동으로 포함되진 않는다.  

아래 코드는 spring.redis 설정(RedisProperties.class)이 지정되면 auto-configure 되는 `RedisAutoConfiguration` 이라는 클래스이다. 

{% highlight java %}
@AutoConfiguration
@ConditionalOnClass(RedisOperations.class)
@EnableConfigurationProperties(RedisProperties.class)
@Import({ LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class })
public class RedisAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean(name = "redisTemplate")
    @ConditionalOnSingleCandidate(RedisConnectionFactory.class)
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<Object, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }
    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnSingleCandidate(RedisConnectionFactory.class)
        public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory redisConnectionFactory) {
        return new StringRedisTemplate(redisConnectionFactory);
    }
}
{% endhighlight %}

4번째 라인을 보면 각 설정을 `@Impport` 하고있는데, lettuce 코드를 보면 다음과 같다.

{% highlight java %}
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RedisClient.class)
@ConditionalOnProperty(name = "spring.redis.client-type", havingValue = "lettuce", matchIfMissing = true)
class LettuceConnectionConfiguration extends RedisConnectionConfiguration {
    LettuceConnectionConfiguration(RedisProperties properties,
        ObjectProvider<RedisStandaloneConfiguration> standaloneConfigurationProvider,
        ObjectProvider<RedisSentinelConfiguration> sentinelConfigurationProvider,
        ObjectProvider<RedisClusterConfiguration> clusterConfigurationProvider) {

        super(properties, standaloneConfigurationProvider, sentinelConfigurationProvider, clusterConfigurationProvider);
    }
// ...
{% endhighlight %}

결국 lettuce 값이 있으면 해당 클래스가 적용되게 된다.  
저기서 부모 클래스를 확인해보면..

{% highlight java %}
abstract class RedisConnectionConfiguration {
    private static final boolean COMMONS_POOL2_AVAILABLE = ClassUtils.isPresent("org.apache.commons.pool2.ObjectPool",
    RedisConnectionConfiguration.class.getClassLoader());
    private final RedisProperties properties;
    private final RedisStandaloneConfiguration standaloneConfiguration;
    private final RedisSentinelConfiguration sentinelConfiguration;
    private final RedisClusterConfiguration clusterConfiguration;
//...
{% endhighlight %}

`org.apache.commons.pool2.ObjectPool` 클래스가 존재해야 설정이 적용되는걸 알 수 있다.  

{% highlight java %}
protected boolean isPoolEnabled(Pool pool) {
    Boolean enabled = pool.getEnabled();
    return (enabled != null) ? enabled : COMMONS_POOL2_AVAILABLE;
}
{% endhighlight %}

{% highlight java %}
private LettuceClientConfigurationBuilder createBuilder(Pool pool) {
    if (isPoolEnabled(pool)) {
        return new PoolBuilderFactory().createBuilder(pool);
    }
    return LettuceClientConfiguration.builder();
}
{% endhighlight %}

조건을 만족해야만 pool이 생성되므로, 해당 패키지가 없다면 connection pool이 생성되지 않는다.  

```yaml
// redis connection pool
implementation "org.apache.commons:commons-pool2:2.12.0"
```
위 설정을 추가로 넣어 주어야 connection pool이 설정되게 된다.  


## 주의/고려사항
connection pool을 설정했다고 하더라도, redis의 non-transaction 명령어에 대해서는 1개의 connection pool만 여러 thread가 돌아가며 사용하게 된다.  

[spring 문서](https://docs.spring.io/spring-data/redis/docs/3.1.9/reference/html/#redis:connectors:lettuce) 를 참고하면,  
> There are also a few Lettuce-specific connection parameters that can be tweaked. By default, all LettuceConnection instances created by the LettuceConnectionFactory share the same thread-safe native connection for all non-blocking and non-transactional operations. To use a dedicated connection each time, set shareNativeConnection to false. LettuceConnectionFactory can also be configured to use a LettucePool for pooling blocking and transactional connections or all connections if shareNativeConnection is set to false.

shareNativeConnection을 false로 해야지만 connection을 공유하지 않고 새로 사용한다고 되어 있다.  
(default: true)  

관련해서 좀 찾아보다가 redis 개발자가 남긴 comment를 발견했다.  
[링크](https://github.com/spring-projects/spring-boot/issues/14196#issuecomment-416489631)

> We should clarify one thing first before extending configuration: Redis itself is single-threaded so increasing the connection count for Redis Standalone does not improve throughput performance.

redis를 standalone으로 사용하는 경우 redis에서 값을 변화시키는 작업은 어짜피 single thread라서 connection 개수를 늘리더라도 성능이 향상되지 않는다는 말이다.  

redis에서 transaction을 보장하는 방식은 다음과 같다.  
[공식문서](https://redis.io/docs/latest/develop/interact/transactions/) 참고

```
> MULTI
OK
> INCR foo
QUEUED
> INCR bar
QUEUED
> EXEC
1) (integer) 1
2) (integer) 1
```

위처럼 blocking 명령어를 사용하는 경우에만 동일한 connection이 아닌 새로운 connection을 이용해서 사용하게 된다.  
`MULTI` 명령어를 사용한 뒤 `EXEC`를 사용하지 않으면 RDB의 경우 `commit` 명령어를 날리지 않고 명령어를 수행한것과 유사하다.  

redis에서 동일 connection을 공유하여 사용하는데 `MULTI` 명령어를 사용하게 될 경우 다른 thread에서 보낸 요청도 `EXEC` 전까지 모두 blocking된다.  
따라서 transaction을 사용할 경우에만 connection을 다르게 사용하고, 일반 명령어를 사용할 경우에는 connection을 공유해도 크게 상관이 없고, connection을 1개만 유지해도 되므로 오히려 자원 효율적이다.  

아래 코드는 spring에서 redis connection을 획득하는 과정인데, 결국 MULTI일때만 다른 connection을 생성하고, 아닌 경우 connection을 재사용한다.  
{% highlight java %}
RedisClusterAsyncCommands<byte[], byte[]> getAsyncConnection() {
    if (this.isQueueing()) {  
        return this.getAsyncDedicatedConnection();
    } else {
        return (this.asyncSharedConn != null && this.asyncSharedConn instanceof StatefulRedisConnection ?
            ((StatefulRedisConnection)this.asyncSharedConn).async() : this.getAsyncDedicatedConnection());
    }
}
public boolean isQueueing() {
    return this.isMulti;
}
public void multi() {
// ...
    this.isMulti = true;
// ...
}
{% endhighlight %}

## 정리
* non-transaction 명령어는 connection pool이 크더라도 한개만 사용된다.
* connection pool은 redis transaction을 사용할때만 새로운 connection이 할당된다.
* redis transaction을 사용하지 않거나 자주 사용하지 않는다면 connection pool을 설정할 필요도 없고, 크게 잡을 이유도 없다.

## 그 외에..
lettuce 코드를 쭉 보다보니 다음과 같은 옵션을 발견했다.  
client 옵션 중에 readFrom 이라는 값이 있는데, 어디서 값을 읽는데 사용할것이냐에 대한 것이다.  
[링크](https://github.com/redis/lettuce/wiki/ReadFrom-Settings) 참고  

아래를 잘 구성하면 좀 더 좋은 방향으로 코드를 작성할 수 있을 듯..

{% highlight java %}
public static final ReadFrom MASTER = new ReadFromImpl.ReadFromUpstream();
public static final ReadFrom MASTER_PREFERRED = new ReadFromImpl.ReadFromUpstreamPreferred();
public static final ReadFrom UPSTREAM = new ReadFromImpl.ReadFromUpstream();
public static final ReadFrom UPSTREAM_PREFERRED = new ReadFromImpl.ReadFromUpstreamPreferred();
public static final ReadFrom REPLICA_PREFERRED = new ReadFromImpl.ReadFromReplicaPreferred();
@Deprecated
public static final ReadFrom SLAVE_PREFERRED = REPLICA_PREFERRED;
public static final ReadFrom REPLICA = new ReadFromImpl.ReadFromReplica();
@Deprecated
public static final ReadFrom SLAVE = REPLICA;
public static final ReadFrom LOWEST_LATENCY = new ReadFromImpl.ReadFromLowestCommandLatency();
@Deprecated
public static final ReadFrom NEAREST = LOWEST_LATENCY;
public static final ReadFrom ANY = new ReadFromImpl.ReadFromAnyNode();
public static final ReadFrom ANY_REPLICA = new ReadFromImpl.ReadFromAnyReplica();
{% endhighlight %}

## 테스트해봤던 설정들 예시
{% highlight java %}
@Slf4j
@Configuration
@RequiredArgsConstructor
public class RedisConfig implements Serializable {
    private final ObjectMapper bigDecimalObjectMapper;

    @Bean
    @ConfigurationProperties(prefix = "application.redis.test")
    public RedisProperties testRedis() {
        return new RedisProperties();
    }

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        final RedisProperties test = this.testRedis();

        RedisStandaloneConfiguration serverConfig = this.createServerConfig(test);
        LettuceClientConfiguration clientConfig = this.createClientConfig(test);

        return new LettuceConnectionFactory(serverConfig, clientConfig);
    }

    @Bean
    public RedisTemplate<?, ?> redisTemplate() {
        Jackson2JsonRedisSerializer<Object> jsonSerializer = new Jackson2JsonRedisSerializer<>(Object.class);
        jsonSerializer.setObjectMapper(this.bigDecimalObjectMapper);

        RedisTemplate<?, ?> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(this.redisConnectionFactory());
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(jsonSerializer);
        redisTemplate.setHashKeySerializer(new StringRedisSerializer());
        redisTemplate.setHashValueSerializer(jsonSerializer);

        return redisTemplate;
    }

    private RedisStandaloneConfiguration createServerConfig(RedisProperties redisProperties) {
        RedisStandaloneConfiguration serverConfig = new RedisStandaloneConfiguration();
        serverConfig.setHostName(redisProperties.getHost());
        serverConfig.setPort(redisProperties.getPort());
        serverConfig.setDatabase(redisProperties.getDatabase());

        return serverConfig;
    }

    /**
     * 참고 : {@link RedisAutoConfiguration} {@link LettuceConnectionConfiguration#createClientOptions}
     * 대부분의 메소드는 spring boot의 auto-configure 옵션을 참고함
     */
    private LettuceClientConfiguration createClientConfig(RedisProperties redisProperties) {
        ClientOptions.Builder builder = ClientOptions.builder();
        Duration connectTimeout = redisProperties.getConnectTimeout();
        if (connectTimeout != null) {
            builder.socketOptions(SocketOptions.builder().connectTimeout(connectTimeout).build());
        }
        ClientOptions clientOptions = builder.timeoutOptions(TimeoutOptions.enabled()).build();

        RedisProperties.Pool poolProperties = redisProperties.getLettuce().getPool();

        GenericObjectPoolConfig<?> poolConfig = new GenericObjectPoolConfig<>();
        poolConfig.setMaxTotal(poolProperties.getMaxActive());
        poolConfig.setMaxIdle(poolProperties.getMaxIdle());
        poolConfig.setMinIdle(poolProperties.getMinIdle());
        if (poolProperties.getTimeBetweenEvictionRuns() != null) {
            poolConfig.setTimeBetweenEvictionRuns(poolProperties.getTimeBetweenEvictionRuns());
        }
        if (poolProperties.getMaxWait() != null) {
            poolConfig.setMaxWait(poolProperties.getMaxWait());
        }

        return LettucePoolingClientConfiguration.builder()
                .readFrom(ReadFrom.MASTER) // 중요, https://github.com/lettuce-io/lettuce-core/wiki/ReadFrom-Settings 참고
                .clientOptions(clientOptions)
                .poolConfig(poolConfig)
                .build();
    }

}
{% endhighlight %}