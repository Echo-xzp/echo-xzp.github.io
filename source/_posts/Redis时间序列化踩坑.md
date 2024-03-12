---
title: Redis时间序列化踩坑
tags:
  - Java
  - JavaWeb
  - Redis
  - Spring
categories: Java
abbrlink: 46be0e22
date: 2022-08-13 08:26:48
---



<h1>redis序列化json中,localdatetime的反序列化错误</h1>
lz学习redis时，想给以前的项目添加个缓存，把第一次访问到的数据存到redis里面，结果在运行时就遇到了问题。

`Could not read JSON: Cannot construct instance ofjava.time.LocalDateTime`

大概意思就是从redis里面反序列化拿出数据后无法再实体化`localdatetime`了

于是在网上找到方法：[点击查看](https://blog.csdn.net/Tony_zt/article/details/105074792)

在LocalDateTime属性上加上如下两个注解就行：
`@JsonDeserialize(using = LocalDateTimeDeserializer.class)`

`@JsonSerialize(using = LocalDateTimeSerializer.class`

调试的时候发现时间格式是对了，但还是无法解决问题。

lz最后猜测时`RedisConfig`这个配置类里面添加了一些操作，产生了问题，而lz的这个配置类也是网上找的，具体一些操作也不甚了解，最后在[阿里云](https://developer.aliyun.com/article/769874)上找到一个新的配置类，重新导入后，解决了这个问题。

配置类如下：

```
@Configuration
@EnableCaching
public class RedisConfig {

    //过期时间-1天
    private Duration timeToLive = Duration.ofDays(-1);



    /**
     * RedisTemplate 先关配置
     *
     * @param factory
     * @return
     */
    @Bean
    @SuppressWarnings("all")
    public RedisTemplate&lt;String, Object&gt; redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate&lt;String, Object&gt; template = new RedisTemplate&lt;String, Object&gt;();
        template.setConnectionFactory(factory);
        Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer(Object.class);
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);

        //LocalDatetime序列化
        JavaTimeModule timeModule = new JavaTimeModule();
        timeModule.addDeserializer(LocalDate.class,
                new LocalDateDeserializer(DateTimeFormatter.ofPattern("yyyy-MM-dd")));
        timeModule.addDeserializer(LocalDateTime.class,
                new LocalDateTimeDeserializer(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")));
        timeModule.addSerializer(LocalDate.class,
                new LocalDateSerializer(DateTimeFormatter.ofPattern("yyyy-MM-dd")));
        timeModule.addSerializer(LocalDateTime.class,
                new LocalDateTimeSerializer(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")));

        om.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
        om.registerModule(timeModule);

        jackson2JsonRedisSerializer.setObjectMapper(om);
        StringRedisSerializer stringRedisSerializer = new StringRedisSerializer();
        // key采用String的序列化方式
        template.setKeySerializer(stringRedisSerializer);
        // hash的key也采用String的序列化方式
        template.setHashKeySerializer(stringRedisSerializer);
        // value序列化方式采用jackson
        template.setValueSerializer(jackson2JsonRedisSerializer);
        // hash的value序列化方式采用jackson
        template.setHashValueSerializer(jackson2JsonRedisSerializer);
        template.afterPropertiesSet();
        return template;
    }


    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        //默认1
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(timeToLive)
                .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(keySerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(valueSerializer()))
                .disableCachingNullValues();
        RedisCacheManager redisCacheManager = RedisCacheManager.builder(connectionFactory)
                .cacheDefaults(config)
                .transactionAware()
                .build();
        return redisCacheManager;
    }

    /**
     * key 类型
     * @return
     */
    private RedisSerializer&lt;String&gt; keySerializer() {
        return  new StringRedisSerializer();
    }

    /**
     * 值采用JSON序列化
     * @return
     */
    private RedisSerializer&lt;Object&gt; valueSerializer() {
        return new GenericJackson2JsonRedisSerializer();
    }

}
```

