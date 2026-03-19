---
name: "spring-cloud-patterns"
description: "Spring Cloud patterns for microservices: Config Server, Gateway, service discovery, circuit breaker (Resilience4j), Feign clients. Use when user works with distributed Spring Boot services."
---

# Spring Cloud Patterns Skill

Patterns for building resilient microservices with Spring Cloud.

## Version Notes

- Spring Cloud integrations are highly version-sensitive; verify starters, annotations, and property names against the current Spring Boot / Spring Cloud BOM.
- Examples in this skill are pattern fragments, not copy-paste-ready production code.

## When to Use
- Project uses `spring-cloud` dependencies
- Questions about inter-service communication (Feign, RestClient)
- Circuit breaker / retry / timeout configuration
- API Gateway routing and filters
- Distributed configuration or service discovery

---

## Spring Cloud Gateway

```java
// ✅ Programmatic route configuration
@Configuration
public class GatewayConfig {

    @Bean
    public RouteLocator routes(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("order-service", r -> r
                .path("/api/orders/**")
                .filters(f -> f
                    .stripPrefix(1)
                    .addRequestHeader("X-Gateway-Source", "api-gateway")
                    .circuitBreaker(c -> c
                        .setName("order-service")
                        .setFallbackUri("forward:/fallback/orders"))
                    .retry(config -> config.setRetries(2).setStatuses(HttpStatus.BAD_GATEWAY))
                )
                .uri("lb://order-service"))   // lb:// = load-balanced via service discovery
            .build();
    }
}
```

```yaml
# application.yml — declarative routes
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=1
            - name: CircuitBreaker
              args:
                name: userServiceCB
                fallbackUri: forward:/fallback/users
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
```

---

## Resilience4j (Circuit Breaker)

```java
// ✅ GOOD: Circuit breaker with fallback
@Service
@RequiredArgsConstructor
public class OrderService {

    private final InventoryClient inventoryClient;

    @CircuitBreaker(name = "inventory", fallbackMethod = "defaultInventory")
    @TimeLimiter(name = "inventory")
    @Retry(name = "inventory")
    public CompletableFuture<Integer> getStock(Long productId) {
        return CompletableFuture.supplyAsync(
            () -> inventoryClient.getAvailableStock(productId),
            Executors.newVirtualThreadPerTaskExecutor()
        );
    }

    // Fallback must have same signature + Throwable param
    public CompletableFuture<Integer> defaultInventory(Long productId, Throwable ex) {
        log.warn("Inventory service unavailable for product {}: {}", productId, ex.getMessage());
        return CompletableFuture.completedFuture(0);
    }
}
```

**Virtual thread executor note**: Do NOT wrap `Executors.newVirtualThreadPerTaskExecutor()` in `try-with-resources` when returning `CompletableFuture` — `close()` calls `shutdown()` + `awaitTermination()`, which blocks the calling thread until the task completes, making the method effectively synchronous and breaking `@TimeLimiter`. Virtual thread executors are lightweight and garbage-collectible; for fire-and-forget async usage, let them be collected. For bounded lifecycle control in synchronous methods, use `try-with-resources`.

```yaml
# application.yml — Resilience4j config
resilience4j:
  circuitbreaker:
    instances:
      inventory:
        slidingWindowSize: 10
        failureRateThreshold: 50        # Open circuit at 50% failures
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 3
  retry:
    instances:
      inventory:
        maxAttempts: 3
        waitDuration: 500ms
        retryExceptions:
          - java.net.ConnectException
          - feign.RetryableException
  timelimiter:
    instances:
      inventory:
        timeoutDuration: 2s
```

---

## Feign Client

```java
// ✅ GOOD: Feign client with error decoder
@FeignClient(
    name = "inventory-service",
    fallback = InventoryClientFallback.class   // used when Resilience4j circuit breaker is enabled
)
public interface InventoryClient {

    @GetMapping("/api/inventory/{productId}/stock")
    Integer getAvailableStock(@PathVariable Long productId);

    @PostMapping("/api/inventory/reserve")
    ReservationResponse reserve(@RequestBody ReserveRequest request);
}

// Fallback implementation
@Component
public class InventoryClientFallback implements InventoryClient {

    @Override
    public Integer getAvailableStock(Long productId) {
        return 0;
    }

    @Override
    public ReservationResponse reserve(ReserveRequest request) {
        throw new ServiceUnavailableException("Inventory service is unavailable");
    }
}
```

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          default:
            connectTimeout: 2000
            readTimeout: 5000
            loggerLevel: BASIC
      circuitbreaker:
        enabled: true
```

---

## Spring Cloud Config

```yaml
# Config Server bootstrap
spring:
  cloud:
    config:
      server:
        git:
          uri: ${CONFIG_REPO_URI}
          default-label: main
          search-paths: '{application}'
          clone-on-start: true

# Client (in each microservice)
spring:
  config:
    import: "configserver:${CONFIG_SERVER_URI:http://localhost:8888}"
  application:
    name: order-service   # Resolves to order-service.yml in config repo
  cloud:
    config:
      fail-fast: true
      retry:
        max-attempts: 5
```

```java
// ✅ Refresh config without restart
@RefreshScope
@Component
public class FeatureFlags {

    @Value("${feature.new-checkout-flow:false}")
    private boolean newCheckoutFlow;
}
// POST /actuator/refresh triggers re-injection
```

---

## Service Discovery (Eureka)

```yaml
# Eureka Server
spring:
  application:
    name: discovery-server
eureka:
  client:
    register-with-eureka: false
    fetch-registry: false

# Eureka Client (each microservice)
eureka:
  client:
    service-url:
      defaultZone: ${EUREKA_URI:http://localhost:8761/eureka}
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${random.value}
```

---

## Distributed Tracing

```yaml
# application.yml — Micrometer Tracing (Spring Boot 3+)
management:
  tracing:
    sampling:
      probability: 1.0   # 100% in dev; use 0.1 in prod
  zipkin:
    tracing:
      endpoint: ${ZIPKIN_URI:http://localhost:9411/api/v2/spans}

logging:
  pattern:
    level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"
```

---

## Common Patterns

### Aggregator (API Composition)
```java
// ✅ Compose responses from multiple services in parallel
@Service
@RequiredArgsConstructor
public class OrderAggregatorService {

    private final OrderClient orderClient;
    private final UserClient userClient;
    private final InventoryClient inventoryClient;

    public OrderDetailResponse getOrderDetail(Long orderId) {
        OrderResponse order = orderClient.getOrder(orderId);

        // ✅ try-with-resources is correct here — method is synchronous and
        // we need all futures to complete before returning the result
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            CompletableFuture<UserResponse> userFuture =
                CompletableFuture.supplyAsync(() -> userClient.getUser(order.userId()), executor);
            CompletableFuture<List<StockInfo>> stockFuture =
                CompletableFuture.supplyAsync(() -> inventoryClient.getStock(order.itemIds()), executor);

            CompletableFuture.allOf(userFuture, stockFuture).join();

            return new OrderDetailResponse(order, userFuture.join(), stockFuture.join());
        }
    }
}
```

---

## Quick Reference

| Component | Dependency | Purpose |
|---|---|---|
| Gateway | `spring-cloud-starter-gateway` | API Gateway, routing, rate-limiting |
| Circuit Breaker | `resilience4j-spring-boot3` | Fault tolerance |
| Feign | `spring-cloud-starter-openfeign` | Declarative HTTP clients |
| Config | `spring-cloud-config-server/client` | Centralized configuration |
| Discovery | `spring-cloud-starter-netflix-eureka` | Service registry |
| Tracing | `micrometer-tracing-bridge-otel` | Distributed tracing |

---

## Related Skills
- `spring-boot-patterns` - Base Spring Boot patterns
- `spring-actuator-patterns` - Health checks and metrics per service
- `performance-patterns` - Connection pools and async tuning
