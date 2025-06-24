---
title: "SpringBoot+Redis实现接口限流"
date: "2022-11-26"
tags: ['SpringBoot', 'Redis']
categories: ['动手实践']
---

Redis 作为21世纪最流行的缓存中间件，它也能够实现接口限流的作用，本片文章主要记录个人实现过程。

在分布式高并发系统中，常常需要用到 `缓存` 、`降级` 、 `限流`。

- 缓存：缓存的目的是提升系统访问速度和增大系统处理容量
- 降级：降级是当服务出现问题或者影响到核心流程时，需要暂时屏蔽掉，待高峰或者问题解决后再打开
- 限流：限流的目的是通过对并发访问/请求进行限速，或者对一个时间窗口内的请求进行限速来保护系统，一旦达到限制速率则可以拒绝服务、排队或等待、降级等处理

## 准备工作

### Maven 添加依赖
```xml
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<!-- 方式2需要用到 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

### 安装 Redis

本文就不多赘述这一部分了 QAQ

## Spring Boot 中集成 Redis

### 1. 在application配置文件中配置Redis

```properties
# Redis数据库索引（默认为0）
spring.redis.database=0
# Redis服务器地址
spring.redis.host=127.0.0.1
# Redis服务器连接端口
spring.redis.port=6379
# Redis服务器连接密码（默认为空）
spring.redis.password=
# 连接超时时间（毫秒）
spring.redis.timeout=1000

# 连接池最大连接数（使用负值表示没有限制）
spring.redis.jedis.pool.max-active=20
# 连接池最大阻塞等待时间（使用负值表示没有限制）
spring.redis.jedis.pool.max-wait=-1
# 连接池中的最大空闲连接
spring.redis.jedis.pool.max-idle=10
# 连接池中的最小空闲连接
spring.redis.jedis.pool.min-idle=0
```

### 2. 配置RedisTemplate

```java
@Configuration
@EnableCaching
public class RedisConfig extends CachingConfigurerSupport {

    /**
     * RedisTemplate相关配置
     * 使redis支持插入对象
     */
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        // 配置连接工厂
        template.setConnectionFactory(factory);
        // 设置key的序列化器
        template.setKeySerializer(new StringRedisSerializer());
        // 设置value的序列化器
        //使用Jackson 2，将对象序列化为JSON
        Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer(Object.class);
        //Json转对象类，不设置默认的会将Json转成HashMap
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        jackson2JsonRedisSerializer.setObjectMapper(om);
        template.setValueSerializer(jackson2JsonRedisSerializer);
        return template;
    }
}
```

自此，Spring Boot 已完成集成Redis，可以通过依赖注入使用`RedisTemplate`，如下所示：
```java
@Autowired
private RedisTemplate<String, Object> redisTemplate;
```

## 实现限流（方式1）

### 1. 限流思路

- 编写自定义注解，为后续过滤接口提供标识
- 通过IP+方法名作为key，访问次数作为value的方式对某一用户的请求进行标识
- 每次访问的时候判断key是否存在，count是否超过限制的次数
- 若访问超出限制，则通过拦截器直接返回错误信息：请求过于频繁

### 2. 添加自定义注解 `AccessLimit`

```java
@Inherited
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface AccessLimit {

    /**
     * 请求次数的指定时间范围 : 秒数(redis数据过期时间)
     */
    int second() default 60;

    /**
     * 指定second 时间内 : API请求次数
     */
    int maxCount() default 3;
}

```

### 3. 编写拦截器

```java
@Slf4j
@Component
public class AccessLimitInterceptor implements HandlerInterceptor {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        // Handler 是否为 HandlerMethod 实例
        if (handler instanceof HandlerMethod) {
            HandlerMethod handlerMethod = (HandlerMethod) handler;
            // 获取方法
            Method method = handlerMethod.getMethod();
            // 是否有AccessLimit注解
            if (!method.isAnnotationPresent(AccessLimit.class)) {
                return true;
            }
            // 获取注解内容信息
            AccessLimit accessLimit = method.getAnnotation(AccessLimit.class);
            if (accessLimit == null) {
                return true;
            }
            // 获取次数和超时
            int seconds = accessLimit.second();
            int maxCount = accessLimit.maxCount();

            // 存储key
            String key = IpUtil.getIpAddr(request)+method.getName();

            // 从Redis中获取用户已经访问的次数
            try {
                Integer count = (Integer) redisTemplate.opsForValue().get(key);
                System.out.println("已经访问的次数:" + count);

                // Redis不存在用户访问记录
                if (null == count || -1 == count) {
                    redisTemplate.opsForValue().set(key, 1, seconds, TimeUnit.SECONDS);
                    return true;
                }

                if (count < maxCount) {
                    // 访问次数+1
                    redisTemplate.opsForValue().increment(key);
                    return true;
                }

                if (count >= maxCount) {
                    // 超出访问限制
                    response.setContentType("application/json;charset=UTF-8");
                    OutputStream out = response.getOutputStream();
                    out.write("请求过于频繁请稍后再试".getBytes("UTF-8"));
                    out.flush();
                    out.close();
                    log.warn("请求过于频繁请稍后再试");
                    return false;
                }
            }catch (RedisConnectionFailureException e){
                log.error("redis error" + e.getMessage().toString());
                return true;
            }

        }
        return true;
    }
}
```

### 4. 注册拦截器

```java
@Configuration
public class IntercepterConfig implements WebMvcConfigurer {
    @Autowired
    private AccessLimitInterceptor accessLimitInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(accessLimitInterceptor)
                .addPathPatterns("/**") // 拦截路径
                .excludePathPatterns("/user/login"); // 不拦截路径
    }
}
```

### 5. 通过`AccessLimit`注解测试最终效果

```java
@RestController
public class TestController {

    @AccessLimit(second = 60,maxCount = 5)
    @GetMapping("test")
    public String test(){
        return "TEST";
    }
}
```

## 实现限流（方式2）

### 1. 限流注解

首先需要创建一个限流注解，限流将分为两种情况：
1. 针对当前接口的全局性限流，例如该接口可以在 1 分钟内访问 100 次。
2. 针对某一个 IP 地址的限流，例如某个 IP 地址可以在 1 分钟内访问 100 次。

针对这两种情况，我们创建一个枚举类：
```java
public enum LimitType {
    /**
     * 默认策略全局限流
     */
    DEFAULT,
    /**
     * 根据请求者IP进行限流
     */
    IP
}
```

下一步，创建限流注解：

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RateLimiter {
    /**
     * 限流key前缀
     */
    String key() default "rate_limit:";

    /**
     * 限流时间,单位秒
     */
    int time() default 60;

    /**
     * 限流次数
     */
    int count() default 100;

    /**
     * 限流类型
     */
    LimitType limitType() default LimitType.DEFAULT;
}
```

### 2. 定制 RedisTemplate

在 `Spring Boot` 中，我们其实更习惯使用 `Spring Data Redis` 来操作 `Redis`，不过默认的 `RedisTemplate` 有一个小坑，就是序列化用的是 `JdkSerializationRedisSerializer`，不知道小伙伴们有没有注意过，直接用这个序列化工具将来存到 `Redis` 上的 `key` 和 `value` 都会莫名其妙多一些前缀，这就导致你用命令读取的时候可能会出错。

修改 `RedisTemplate` 序列化方案，代码如下：
```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<Object, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(connectionFactory);
        // 使用Jackson2JsonRedisSerialize 替换默认序列化(默认采用的是JDK序列化)
        Jackson2JsonRedisSerializer<Object> jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer<>(Object.class);
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        jackson2JsonRedisSerializer.setObjectMapper(om);
        redisTemplate.setKeySerializer(jackson2JsonRedisSerializer);
        redisTemplate.setValueSerializer(jackson2JsonRedisSerializer);
        redisTemplate.setHashKeySerializer(jackson2JsonRedisSerializer);
        redisTemplate.setHashValueSerializer(jackson2JsonRedisSerializer);
        return redisTemplate;
    }
}
```

### 3. 开发 Lua 脚本

Redis 中的一些原子操作我们可以借助 Lua 脚本来实现，想要调用 Lua 脚本，我们有两种不同的思路：

1. 在 Redis 服务端定义好 Lua 脚本，然后计算出来一个散列值，在 Java 代码中，通过这个散列值锁定要执行哪个 Lua 脚本。
2. 直接在 Java 代码中将 Lua 脚本定义好，然后发送到 Redis 服务端去执行。

Spring Data Redis 中也提供了操作 Lua 脚本的接口，还是比较方便的，所以我们这里就采用第二种方案。

在 resources 目录下新建 lua 文件夹专门用来存放 lua 脚本，脚本内容如下：
```lua
local key = KEYS[1]
local count = tonumber(ARGV[1])
local time = tonumber(ARGV[2])
local current = redis.call('get', key)
if current and tonumber(current) > count then
    return tonumber(current)
end
current = redis.call('incr', key)
if tonumber(current) == 1 then
    redis.call('expire', key, time)
end
return tonumber(current)
```

KEYS 和 ARGV 都是一会调用时候传进来的参数，tonumber 就是把字符串转为数字，redis.call 就是执行具体的 redis 指令，具体流程是这样：
1. 首先获取到传进来的 key 以及 限流的 count 和时间 time。
2. 通过 get 获取到这个 key 对应的值，这个值就是当前时间窗内这个接口可以访问多少次。
3. 如果是第一次访问，此时拿到的结果为 nil，否则拿到的结果应该是一个数字，所以接下来就判断， 如果拿到的结果是一个数字，并且这个数字还大于 count，那就说明已经超过流量限制了，那么直接返回查询的结果即可。
4. 如果拿到的结果为 nil，说明是第一次访问，此时就给当前 key 自增 1，然后设置一个过期时间。
5. 最后把自增 1 后的值返回就可以了。

接下来我们在一个 Bean 中来加载这段 Lua 脚本，如下：
```java
@Bean
public DefaultRedisScript<Long> limitScript() {
    DefaultRedisScript<Long> redisScript = new DefaultRedisScript<>();
    redisScript.setScriptSource(new ResourceScriptSource(new ClassPathResource("lua/limit.lua")));
    redisScript.setResultType(Long.class);
    return redisScript;
}
```

### 4. 注解解析（通过AspectJ自定义切面）

```java
@Aspect
@Component
public class RateLimiterAspect {
    private static final Logger log = LoggerFactory.getLogger(RateLimiterAspect.class);

    @Autowired
    private RedisTemplate<Object, Object> redisTemplate;

    @Autowired
    private RedisScript<Long> limitScript;

    @Before("@annotation(rateLimiter)")
    public void doBefore(JoinPoint point, RateLimiter rateLimiter) throws Throwable {
        String key = rateLimiter.key();
        int time = rateLimiter.time();
        int count = rateLimiter.count();

        String combineKey = getCombineKey(rateLimiter, point);
        List<Object> keys = Collections.singletonList(combineKey);
        try {
            Long number = redisTemplate.execute(limitScript, keys, count, time);
            if (number==null || number.intValue() > count) {
                throw new ServiceException("访问过于频繁，请稍候再试");
            }
            log.info("限制请求'{}',当前请求'{}',缓存key'{}'", count, number.intValue(), key);
        } catch (ServiceException e) {
            throw e;
        } catch (Exception e) {
            throw new RuntimeException("服务器限流异常，请稍候再试");
        }
    }

    public String getCombineKey(RateLimiter rateLimiter, JoinPoint point) {
        StringBuffer stringBuffer = new StringBuffer(rateLimiter.key());
        if (rateLimiter.limitType() == LimitType.IP) {
            stringBuffer.append(IpUtils.getIpAddr(((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getRequest())).append("-");
        }
        MethodSignature signature = (MethodSignature) point.getSignature();
        Method method = signature.getMethod();
        Class<?> targetClass = method.getDeclaringClass();
        stringBuffer.append(targetClass.getName()).append("-").append(method.getName());
        return stringBuffer.toString();
    }
}
```

这个切面就是拦截所有加了 @RateLimiter 注解的方法，在前置通知中对注解进行处理。

1. 首先获取到注解中的 key、time 以及 count 三个参数。
2. 获取一个组合的 key，所谓的组合的 key，就是在注解的 key 属性基础上，再加上方法的完整路径，如果是 IP 模式的话，就再加上 IP 地址。以 IP 模式为例，最终生成的 key 类似这样：rate_limit:127.0.0.1-com.demo.ratelimiter.controller.HelloController-hello（如果不是 IP 模式，那么生成的 key 中就不包含 IP 地址）。
3. 将生成的 key 放到集合中。
4. 通过 redisTemplate.execute 方法取执行一个 Lua 脚本，第一个参数是脚本所封装的对象，第二个参数是 key，对应了脚本中的 KEYS，后面是可变长度的参数，对应了脚本中的 ARGV。
5. 将 Lua 脚本执行的结果与 count 进行比较，如果大于 count，就说明过载了，抛异常就行了。

### 5. 接口测试

```java
@RestController
public class HelloController {
    @GetMapping("/hello")
    @RateLimiter(time = 5,count = 3,limitType = LimitType.IP)
    public String hello() {
        return "hello>>>"+new Date();
    }
}
```

每一个 IP 地址，在 5 秒内只能访问 3 次。