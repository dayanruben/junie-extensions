---
name: "spring-actuator-patterns"
description: "Spring Boot Actuator configuration, custom health indicators, Micrometer metrics. Use when user asks about health checks, /actuator endpoints, metrics, monitoring, or Prometheus integration."
---

# Spring Actuator Patterns Skill

## Scope and Boundaries

- Use this skill for Actuator endpoint configuration, custom health indicators, Micrometer metrics, and Prometheus integration.
- Use `performance-patterns` for general profiling strategy and bottleneck-oriented optimization.
- Use `kubernetes-patterns` for configuring liveness/readiness probe paths in k8s manifests.
- Use `security-audit` for securing actuator endpoints with Spring Security.

## When to Use
- Configuring `/actuator` endpoints for monitoring
- Implementing custom health indicators
- Adding business metrics with Micrometer
- Securing actuator endpoints
- Integrating with Prometheus / Grafana

---

## Endpoint Configuration

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "health,info,metrics,loggers,prometheus"
        # ❌ Don't expose: env, beans, heapdump, shutdown in production
  endpoint:
    health:
      show-details: when-authorized   # 'always' only in dev
      show-components: when-authorized
  info:
    env:
      enabled: true   # Exposes spring.info.* properties

# Build info in /actuator/info
info:
  app:
    name: ${spring.application.name}
    version: @project.version@
```

---

## Securing Actuator Endpoints

```java
// ✅ Secure within existing SecurityFilterChain (simpler — no separate chain needed)
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/actuator/health", "/actuator/info").permitAll()
    .requestMatchers("/actuator/**").hasRole("ADMIN")
    .anyRequest().authenticated()
)
```

---

## Custom Health Indicator

```java
// ✅ Report health of an external dependency
@Component
public class PaymentGatewayHealthIndicator implements HealthIndicator {

    private final PaymentGatewayClient client;

    @Override
    public Health health() {
        try {
            boolean reachable = client.ping();
            return reachable
                ? Health.up().withDetail("gateway", "reachable").build()
                : Health.down().withDetail("gateway", "timeout").build();
        } catch (Exception e) {
            return Health.down(e).withDetail("gateway", "error").build();
        }
    }
}
```

---

## Micrometer Metrics

```kotlin
@Service
class OrderService(meterRegistry: MeterRegistry) {

    // Counter — total events
    private val orderCounter = meterRegistry.counter("orders.created", "env", "prod")

    // Timer — latency + count + total time
    private val orderTimer = meterRegistry.timer("orders.processing.duration")

    fun createOrder(request: CreateOrderRequest): Order =
        orderTimer.recordCallable {
            processOrder(request).also {
                orderCounter.increment()
            }
        }!!
}
```

```java
// ✅ @Timed annotation (simpler, AOP-based)
@Timed(value = "orders.create", description = "Time to create order")
public Order createOrder(CreateOrderRequest request) { ... }
```

---

## Prometheus Integration

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "health,prometheus"
  metrics:
    tags:
      application: ${spring.application.name}   # Common tag on all metrics
```

Metrics available at `/actuator/prometheus` — scrape with Prometheus, visualize in Grafana.

---

## Liveness and Readiness Probes (Kubernetes)

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
```

Kubernetes probe paths:
- Liveness: `GET /actuator/health/liveness`
- Readiness: `GET /actuator/health/readiness`

---

## Anti-Patterns

- Exposing all actuator endpoints (`include: "*"`) to the public network without security — use `hasRole("ADMIN")` or a separate management port.
- Not setting `management.server.port` to separate management traffic from application traffic in production.
- Custom health indicators that make slow external calls without timeouts — a hanging health check blocks readiness and can cause cascading failures.
- Missing `@Timed` on critical endpoints — no latency visibility until an incident.
- Using `show-details: always` in production — leaks internal infrastructure details (database URLs, broker addresses).
