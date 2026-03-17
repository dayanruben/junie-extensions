---
name: "performance-patterns"
description: "JVM performance patterns: caching, connection pools, Micrometer metrics, profiling hints. Use when optimizing application performance."
---

# Performance Patterns Skill

Use this skill when performance is already the primary problem and you need measurement-first guidance for latency, throughput, CPU, memory, or cache behavior.

## Scope and Boundaries

- Use this skill for profiling strategy, caching, pool sizing, performance smells, and bottleneck-oriented optimization.
- Use `debugging-investigation-patterns` first if the root cause is still unknown.
- Use `jpa-patterns` for ORM-specific fetch and transaction issues.
- Use `spring-actuator-patterns` for detailed Actuator and Micrometer setup.
- Treat examples as pattern fragments; validate concrete numbers against the current workload.

## When to Use
- Optimizing slow endpoints, jobs, or queries
- Investigating high CPU, memory pressure, or throughput collapse
- Reviewing caching strategies
- Validating whether a suspected hotspot is real

## Performance Workflow

1. Reproduce the slowdown with a representative scenario.
2. Measure before changing code.
   - latency percentiles
   - throughput
   - error rate under load
   - CPU, memory, GC, and pool saturation
3. Identify the dominant bottleneck.
   - database or remote I/O
   - lock contention
   - allocation / GC churn
   - inefficient algorithm or data structure
4. Fix the most expensive bottleneck first.
5. Re-measure and keep the change only if the result is material.

## Code-Level Smells Worth Checking

- Repeated database or HTTP calls inside loops
- Building large intermediate collections when streaming or batching would be enough
- Regex compilation, JSON mapper creation, or expensive object creation on hot paths
- Excessive boxing, copying, or conversion between collection types in tight loops
- Caching before measuring whether the repeated work is actually expensive

## Caching

For Spring Cache with Redis (`@Cacheable`, `@CacheEvict`, TTL, stampede prevention), see `redis-patterns`.

Key performance considerations for caching:
- Measure that the repeated work is actually expensive before adding a cache.
- Set a memory budget and TTL — unbounded caches become memory leaks.
- Use TTL jitter to prevent stampede (all keys expiring simultaneously).
- Invalidation strategy must be clear: who evicts, when, and what happens on stale reads.
- Monitor cache hit ratio — a cache with < 50% hit rate may not be worth the complexity.

---

## Connection Pool Tuning (HikariCP)

- Treat pool size formulas as a starting point, not a law.
- Increase pool sizes only when you have evidence of saturation and enough downstream capacity.
- Watch for queueing, long checkout times, and database-side contention.

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10        # CPU cores * 2 + disk spindles
      minimum-idle: 5
      connection-timeout: 30000    # 30s
      idle-timeout: 600000         # 10min
      max-lifetime: 1800000        # 30min
      leak-detection-threshold: 60000
```

---

## Micrometer Metrics

For Micrometer setup, custom metrics, and Prometheus integration see `spring-actuator-patterns`.

Use metrics to confirm bottlenecks before and after optimization — measure latency percentiles and throughput, not just averages.

---

## Async Processing

- Do not offload work to background threads only to hide slow synchronous logic.
- Prefer bounded executors and explicit backpressure over unbounded fan-out.
- Measure queue growth and end-to-end latency, not just method execution time.

```kotlin
// ✅ Good — offload heavy work to background
@Service
class EmailService {

    @Async
    fun sendWelcomeEmail(user: User) {
        // runs in thread pool, doesn't block HTTP thread
        emailClient.send(user.email, "Welcome!")
    }
}
```

## Review Checklist

- Do we have a reproducible scenario and baseline numbers?
- Is the bottleneck confirmed rather than assumed?
- Are we optimizing I/O, allocation, locking, or algorithmic complexity?
- Is the cache strategy bounded, invalidated, and justified by measurements?
- Are pool sizes and async settings validated against the downstream system?

## Anti-Patterns

- Micro-optimizing code before finding the dominant bottleneck
- Adding caches without ownership, invalidation, or memory budget
- Increasing thread or connection pools to compensate for slow dependencies blindly
- Reporting average latency only and ignoring tail latency
