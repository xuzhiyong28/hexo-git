---
title: 基于springBoot-aop实现redis分布式限流
tags:
  - redis
  - 项目实践
categories: redis
description : 基于springBoot+redis+aop+自定义标签实现分布式限流
date: 2019-02-18 17:14:00
---
<!--more-->

## 项目介绍

### 原理
通过给api-url设置一个redis-key并设置过期时间。当每次url被请求时key的值加一，如果在过期时间内增加到一定数量后则进行限流。在原理上通过使用redis键值过期时间作为限流的单位时间，在单位时间内如果访问的次数大于指定次数后禁止继续访问，当键值过期后再次访问时重新计算单位时间内的允许访问次数。
<!--more-->
采用aop+自定义注解的方式，在每次调用controller类时对方法进行增强，aop扫描所有注有**@RateLimiter**的方法并环绕增强。
### 项目测试

```java
//指定testone这个url在10秒内只能有5次请求，如果超过则报错
@RequestMapping("/testone")
@RateLimiter(key = "ratelimit:testOne", limit = 5, expire = 10, message = MESSAGE, isLimiterIp = true)
    public String testOne(HttpServletRequest request) {
        return "正常请求";
}
```

连续在10秒内访问5次

```json
{"code":"400","msg":"FAIL","desc":"触发限流"}
```

## 代码实现

### MAVEN包
```xml
<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<!--aop支持-->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-aop</artifactId>
		</dependency>
		<!--添加redis支持-->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-redis</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-devtools</artifactId>
			<scope>runtime</scope>
			<optional>true</optional>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>org.apache.commons</groupId>
			<artifactId>commons-pool2</artifactId>
			<version>2.6.0</version>
		</dependency>
		<dependency>
			<groupId>org.apache.commons</groupId>
			<artifactId>commons-lang3</artifactId>
			<version>3.7</version>
		</dependency>

		<dependency>
			<groupId>com.google.guava</groupId>
			<artifactId>guava</artifactId>
			<version>26.0-jre</version>
		</dependency>

	</dependencies>
````

### SpringBoot配置

```yaml
spring:
  redis:
    database: 0
    host: 127.0.0.1
    jedis:
      pool:
        #最大连接数据库连接数,设 0 为没有限制
        max-active: 8
        #最大等待连接中的数量,设 0 为没有限制
        max-idle: 8
        #最大建立连接等待时间。如果超过此时间将接到异常。设为-1表示无限制。
        max-wait: -1ms
        #最小等待连接中的数量,设 0 为没有限制
        min-idle: 0
    lettuce:
      pool:
        max-active: 8
        max-idle: 8
        max-wait: -1ms
        min-idle: 0
      shutdown-timeout: 100ms
    password: ''
    port: 6379
```

### 代码
#### 自定义注解

```java
package com.xzy.springbootredisdemo.annotation;

import java.lang.annotation.*;

/***
 * 限流注解
 * key--表示限流模块名，指定该值用于区分不同应用，不同场景，推荐格式为：应用名:模块名:ip:接口名:方法名
 * limit--表示单位时间允许通过的请求数
 * expire--incr的值的过期时间，业务中表示限流的单位时间。
 * isLimiterIp -- 是否根据IP进行过滤，如果为true，则key为ratelimit:url:ip的方式存储，这样可以根据ip来限流
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RateLimiter {
    /**
     * 限流key
     * @return
     */
    String key() default "rate:limiter";
    /**
     * 单位时间限制通过请求数
     * @return
     */
    long limit() default 10;
    /**
     * 过期时间，单位秒
     * @return
     */
    long expire() default 1;
    String message() default "限制访问";
    /***
     * 是否精确到ip控制
     * * @return
     */
    boolean isLimiterIp() default false;
}

```
#### Redis配置类

```java
import com.fasterxml.jackson.annotation.JsonAutoDetect;
import com.fasterxml.jackson.annotation.PropertyAccessor;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.CachingConfigurerSupport;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.RedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;
import java.time.Duration;
@Configuration
@EnableCaching
public class RedisCacheConfig extends CachingConfigurerSupport {
    private static final Logger logger = LoggerFactory.getLogger(RedisCacheConfig.class);
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory) {
        RedisSerializer<String> redisSerializer = new StringRedisSerializer();
        Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer(Object.class);
        //解决查询缓存转换异常的问题
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        jackson2JsonRedisSerializer.setObjectMapper(om);
        // 配置序列化（解决乱码的问题）,过期时间30秒
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofSeconds(30))
                .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(redisSerializer))
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(jackson2JsonRedisSerializer))
                .disableCachingNullValues();
        RedisCacheManager cacheManager = RedisCacheManager.builder(factory)
                .cacheDefaults(config)
                .build();
        return cacheManager;
    }
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        RedisSerializer<String> redisSerializer = new StringRedisSerializer();
        Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer(Object.class);
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        jackson2JsonRedisSerializer.setObjectMapper(om);
        template.setConnectionFactory(factory);
        //key序列化方式
        template.setKeySerializer(redisSerializer);
        //value序列化
        template.setValueSerializer(jackson2JsonRedisSerializer);
        //value hashmap序列化
        template.setHashValueSerializer(jackson2JsonRedisSerializer);
        return template;
    }
}
```

#### AOP增强

对使用了@RateLimiter注解的方法进行环绕增强

```java
import com.google.common.base.Preconditions;
import com.google.common.collect.Lists;
import com.google.common.collect.Maps;
import com.xzy.springbootredisdemo.annotation.RateLimiter;
import com.xzy.springbootredisdemo.util.IpUtil;
import com.xzy.springbootredisdemo.util.ResultVo;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.Signature;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.io.ClassPathResource;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import org.springframework.scripting.support.ResourceScriptSource;
import org.springframework.stereotype.Component;

import javax.annotation.PostConstruct;
import javax.servlet.http.HttpServletRequest;
import java.lang.reflect.Method;
import java.lang.reflect.Type;
import java.util.List;
import java.util.Map;

//aop支持
@Aspect
@Component
public class RateLimterHandler {
    private static final Logger logger = LoggerFactory.getLogger(RateLimterHandler.class);

    @Autowired
    private RedisTemplate redisTemplate;
    
    private DefaultRedisScript<Long> redisScript;

    //@PostConstruct修饰的方法会在服务器加载Servlet的时候运行，并且只会被服务器调用一次，类似于Serclet的inti()方法
    @PostConstruct
    public void init() {
        redisScript = new DefaultRedisScript<>();
        redisScript.setResultType(Long.class);
        redisScript.setScriptSource(new ResourceScriptSource(new ClassPathResource("rateLimter.lua")));
        logger.info("RateLimterHandler[分布式限流处理器]脚本加载完成");
    }

    //定义切点方法为注解RateLimiter名称为rateLimiter
    @Pointcut("@annotation(com.xzy.springbootredisdemo.annotation.RateLimiter)")
    public void rateLimiter() {
    }

    @Around("@annotation(rateLimiter)")
    public Object around(ProceedingJoinPoint proceedingJoinPoint, RateLimiter rateLimiter) throws Throwable {
        Signature signature = proceedingJoinPoint.getSignature();
        if (!(signature instanceof MethodSignature)) {
            throw new IllegalArgumentException("注释应该配置在方法上");
        }
        //获取方法参数
        HttpServletRequest request = getArgsRequest(proceedingJoinPoint);
        //获取注解参数
        // 限流模块key
        String limitKey = rateLimiter.key();
        Preconditions.checkNotNull(limitKey); //判断是否为空
        //是否根据ip进行限流
        boolean isLimiterIp = rateLimiter.isLimiterIp();
        if (isLimiterIp && request != null) {
            if (request != null) {
                String ip = IpUtil.getIpAddr(request);
                limitKey += ":" + ip;
            }
        }
        //限流阀值
        long limitTimes = rateLimiter.limit();
        // 限流超时时间
        long expireTime = rateLimiter.expire();
        if (logger.isDebugEnabled()) {
            logger.debug("RateLimterHandler[分布式限流处理器]参数值为-limitTimes={},limitTimeout={}", limitTimes, expireTime);
        }
        //限流提示语
        String message = rateLimiter.message();
        //执行lua脚本
        List<String> keyList = Lists.newArrayList();
        keyList.add(limitKey);
        Long result = (Long) redisTemplate.execute(redisScript, keyList, expireTime, limitTimes);
        if (result == 0) {
            Type type = getMethodReturnType(proceedingJoinPoint);
            logger.info("由于超过单位时间=" + expireTime + "-允许的请求次数=" + limitTimes + "[触发限流]");
            return limitErrorReturn(type, message);
        }
        return proceedingJoinPoint.proceed();
    }


    public Object limitErrorReturn(Type type, String msg) {
        switch (type.getTypeName()) {
            case "java.lang.String":
                return msg;
            case "java.util.Map":
                Map<String, String> resultMap = Maps.newHashMap();
                resultMap.put("code", "-1");
                resultMap.put("message", msg);
                return resultMap;
            case "com.xzy.springbootredisdemo.util.ResultVo":
                return ResultVo.getErrorResultVo(msg);
            default:
                return msg;
        }
    }


    public HttpServletRequest getArgsRequest(ProceedingJoinPoint proceedingJoinPoint) throws NoSuchMethodException {
        //获取方法参数
        Object[] args = proceedingJoinPoint.getArgs();
        HttpServletRequest request = null;
        for (Object o : args) {
            if (o instanceof HttpServletRequest) {
                request = (HttpServletRequest) o;
            }
        }
        return request;
    }

    /***
     * 获取返回值类型
     * @param proceedingJoinPoint
     * @return
     * @throws NoSuchMethodException
     */
    public Type getMethodReturnType(ProceedingJoinPoint proceedingJoinPoint) throws NoSuchMethodException {
        Signature signature = proceedingJoinPoint.getSignature();
        MethodSignature methodSignature = (MethodSignature) signature;
        Object target = proceedingJoinPoint.getTarget();
        Method currentMethod = target.getClass().getMethod(methodSignature.getName(), methodSignature.getParameterTypes());
        return currentMethod.getAnnotatedReturnType().getType();
    }

}

```

#### 枚举类

```java
public interface BaseEnum {
    public String getCode();
    public String getMsg();
}
public enum SystemResultEnum implements BaseEnum {
    SUCCESS("1", "成功"),
    FAIL("0", "系统错误"),
    NO_AUTH_USER("2", "未注册用户"),
    NO_AD_USER("3", "待审核用户");
    SystemResultEnum(String code, String msg) {
        this.code = code;
        this.msg = msg;
    }
    private String code;
    private String msg;
    public String getCode() {
        return code;
    }
    public void setCode(String code) {
        this.code = code;
    }
    public String getMsg() {
        return msg;
    }
    public void setMsg(String msg) {
        this.msg = msg;
    }
}
```

#### Lua脚本

```lua
-- 获取KEY
local key1 = KEYS[1]
-- incr可以实现自动设置key并设置初始值为1
local val = redis.call('incr', key1)
-- 获取key1剩下的时间
local ttl = redis.call('ttl',key1)

--获取ARGV内的参数并打印
-- ARGV[1]超时时间
local expire = ARGV[1]
-- ARGV[2]单位时间允许通过的请求数
local times = ARGV[2]

-- 日志打印 没什么用
-- redis.log(redis.LOG_DEBUG,tostring(times))
-- redis.log(redis.LOG_DEBUG,tostring(expire))
-- redis.log(redis.LOG_NOTICE, "incr "..key1.." "..val);
-- val == 1 表示是第一次访问，此时是第一次访问，设置过期时间
if val == 1 then
    redis.call('expire',key1,tonumber(expire))
else
    if ttl == -1 then
        redis.call('expire',key1,tonumber(expire))
    end
end

if val > tonumber(times) then
    return 0
end
    return 1
```



#### 控制层

```java
@RestController
public class TestController {
    private static final String MESSAGE = "{\"code\":\"400\",\"msg\":\"FAIL\",\"desc\":\"触发限流\"}";

    @RequestMapping("/testone")
    @RateLimiter(key = "ratelimit:testOne", limit = 5, expire = 10, message = MESSAGE, isLimiterIp = true)
    public String testOne(HttpServletRequest request) {
        return "正常请求";
    }
    @RequestMapping("/testtwo")
    @RateLimiter(key = "ratelimit:testTwo", limit = 5, expire = 10, isLimiterIp = false)
    public Map testTow(HttpServletRequest request){
        Map<String,String> resultMap = Maps.newHashMap();
        resultMap.put("code", "1");
        resultMap.put("message" , "success");
        return resultMap;
    }
    @RequestMapping("/testthree")
    @RateLimiter(key = "ratelimit:testThree", limit = 5, expire = 10, isLimiterIp = false)
    public ResultVo testThree(HttpServletRequest request){
        return ResultVo.getSuccessResultVo();
    }
}
```