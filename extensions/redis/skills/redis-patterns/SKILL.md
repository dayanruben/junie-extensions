---
name: "redis-patterns"
description: "Redis integration with Spring Boot: Spring Cache, RedisTemplate, session, TTL, serialization. Use when caching with Redis, managing sessions, or working with Redis data structures."
---

# Redis Patterns Skill

Best practices for using Redis with Spring Boot and Spring Data Redis.

## Scope and Boundaries

- Use this skill for Spring Cache with Redis, RedisTemplate, sessions, distributed locks, and TTL strategies.
- Use `performance-patterns` for general caching strategy decisions (when to cache, measurement workflow).
- Use `spring-boot-patterns` for general service and configuration patterns.
- Examples use Spring Data Redis with Lettuce (default driver in Spring Boot).

## When to Use
- Adding caching layer with Redis
- Managing HTTP sessions in Redis
- Working with RedisTemplate for custom operations
- Setting up distributed locks or rate limiters

---

## Spring Cache with Redis (Most Common)

```java
// 1. Enable caching
@SpringBootApplication
@EnableCaching
public class Application { ... }
```

```yaml
# application.yml
spring:
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: 6379
      password: ${REDIS_PASSWORD:}
  cache:
    redis:
      time-to-live: 600s        # Default TTL for all caches
      cache-null-values: false  # Don't cache null results
```

```java
// 2. Use on service methods
@Service
@RequiredArgsConstructor
public class UserService {

    @Cacheable(value = "users", key = "#id")
    public User findById(Long id) {
        return userRepository.findById(id).orElseThrow();
    }

    @CachePut(value = "users", key = "#result.id")
    public User update(UpdateUserRequest request) {
        return userRepository.save(mapper.toUser(request));
    }

    @CacheEvict(value = "users", key = "#id")
    public void delete(Long id) {
        userRepository.deleteById(id);
    }

    // Evict all entries in a cache
    @CacheEvict(value = "users", allEntries = true)
    public void clearCache() {}
}
```

---

## RedisTemplate — Custom Operations

```java
// ✅ Proper serialization configuration
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class SessionStore {

    private final RedisTemplate<String, Object> redisTemplate;

    // ✅ Set with TTL
    public void store(String key, Object value, Duration ttl) {
        redisTemplate.opsForValue().set(key, value, ttl);
    }

    // ✅ Get
    public Optional<Object> get(String key) {
        return Optional.ofNullable(redisTemplate.opsForValue().get(key));
    }

    // ✅ Atomic increment (counter, rate limiter)
    public Long increment(String key) {
        return redisTemplate.opsForValue().increment(key);
    }

    // ✅ Hash operations (partial updates)
    public void setField(String key, String field, Object value) {
        redisTemplate.opsForHash().put(key, field, value);
    }
}
```

---

## TTL and Cache Stampede

```java
// ✅ Per-cache TTL configuration
@Bean
public RedisCacheManagerBuilderCustomizer redisCacheManagerBuilderCustomizer() {
    return builder -> builder
        .withCacheConfiguration("users",
            RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10))
                .disableCachingNullValues())
        .withCacheConfiguration("products",
            RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofHours(1)));
}
```

```java
// ✅ TTL with jitter — prevents cache stampede (all keys expiring simultaneously)
public void storeWithJitter(String key, Object value, Duration baseTtl) {
    long jitter = ThreadLocalRandom.current().nextLong(0, baseTtl.toSeconds() / 10);
    redisTemplate.opsForValue().set(key, value, baseTtl.plusSeconds(jitter));
}
```

---

## Spring Session (HTTP Session in Redis)

```kotlin
// build.gradle.kts
implementation("org.springframework.session:spring-session-data-redis")
```

```java
@Configuration
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)  // 30 min
public class SessionConfig {}
```
Sessions are automatically stored in Redis — no other changes needed in controllers.

---

## Distributed Lock

```java
// ✅ Simple distributed lock with SETNX + TTL
@Service
@RequiredArgsConstructor
public class DistributedLock {

    private final StringRedisTemplate redisTemplate;

    public boolean tryLock(String lockKey, String requestId, Duration ttl) {
        Boolean acquired = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, requestId, ttl);
        return Boolean.TRUE.equals(acquired);
    }

    public void unlock(String lockKey, String requestId) {
        // Only delete if we own the lock (prevents releasing someone else's lock)
        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                        "return redis.call('del', KEYS[1]) else return 0 end";
        redisTemplate.execute(
            RedisScript.of(script, Long.class),
            List.of(lockKey),
            requestId
        );
    }
}
```

---

## Common Mistakes

```java
// ❌ Default Java serialization — breaks between deployments, security risk
template.setValueSerializer(new JdkSerializationRedisSerializer());

// ✅ JSON serialization
template.setValueSerializer(new GenericJackson2JsonRedisSerializer());

// ❌ No TTL — keys accumulate forever
redisTemplate.opsForValue().set(key, value);

// ✅ Always set TTL
redisTemplate.opsForValue().set(key, value, Duration.ofMinutes(30));

// ❌ @Cacheable on private methods or same-class calls — AOP proxy bypassed
@Service
public class MyService {
    @Cacheable("items")
    private Item internalMethod(Long id) { ... }  // won't work
}
```

---

## Quick Reference

| Need | Solution |
|---|---|
| Simple caching | `@Cacheable` / `@CacheEvict` |
| Custom TTL per cache | `RedisCacheManagerBuilderCustomizer` |
| Session management | `spring-session-data-redis` |
| Raw operations | `RedisTemplate` |
| Atomic counter | `opsForValue().increment()` |
| Distributed lock | `setIfAbsent()` + Lua script for unlock |
| Prevent stampede | TTL jitter |
