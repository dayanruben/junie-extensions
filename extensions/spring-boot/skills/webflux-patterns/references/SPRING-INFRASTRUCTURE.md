# Spring WebFlux Infrastructure Patterns

Spring-specific infrastructure: reactive controllers, WebClient, SSE, reactive security, R2DBC, and common anti-patterns for mixing blocking/reactive code.

---

## Reactive Controller

```java
@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
public class ProductController {

    private final ProductService productService;

    @GetMapping
    public Flux<ProductResponse> getAll() {
        return productService.findAll();
    }

    @GetMapping("/{id}")
    public Mono<ResponseEntity<ProductResponse>> getById(@PathVariable Long id) {
        return productService.findById(id)
            .map(ResponseEntity::ok)
            .defaultIfEmpty(ResponseEntity.notFound().build());
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono<ProductResponse> create(@Valid @RequestBody Mono<CreateProductRequest> request) {
        return request.flatMap(productService::create);
    }
}
```

### Kotlin (coroutines — preferred in Kotlin projects)
```kotlin
@RestController
@RequestMapping("/api/v1/products")
class ProductController(private val productService: ProductService) {

    @GetMapping
    suspend fun getAll(): List<ProductResponse> = productService.findAll()

    @GetMapping("/{id}")
    suspend fun getById(@PathVariable id: Long): ProductResponse =
        productService.findById(id)

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    suspend fun create(@Valid @RequestBody request: CreateProductRequest): ProductResponse =
        productService.create(request)
}
```

---

## Functional Endpoints (RouterFunction)

Preferred for fine-grained request matching, streaming, or reducing annotation overhead.

```java
@Configuration
public class ProductRouter {

    @Bean
    public RouterFunction<ServerResponse> productRoutes(ProductHandler handler) {
        return RouterFunctions.route()
            .GET("/api/v1/products", handler::getAll)
            .GET("/api/v1/products/{id}", handler::getById)
            .POST("/api/v1/products", handler::create)
            .build();
    }
}

@Component
@RequiredArgsConstructor
public class ProductHandler {

    private final ProductService productService;

    public Mono<ServerResponse> getById(ServerRequest request) {
        Long id = Long.parseLong(request.pathVariable("id"));
        return productService.findById(id)
            .flatMap(p -> ServerResponse.ok().bodyValue(p))
            .switchIfEmpty(ServerResponse.notFound().build());
    }

    public Mono<ServerResponse> create(ServerRequest request) {
        return request.bodyToMono(CreateProductRequest.class)
            .flatMap(productService::create)
            .flatMap(p -> ServerResponse.status(HttpStatus.CREATED).bodyValue(p));
    }
}
```

---

## WebClient — Reactive HTTP Client

Use `WebClient` instead of `RestTemplate` or `Feign` in reactive applications.

```java
// ✅ Bean configuration
@Bean
public WebClient inventoryWebClient(WebClient.Builder builder) {
    return builder
        .baseUrl("http://inventory-service")
        .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
        .codecs(c -> c.defaultCodecs().maxInMemorySize(1 * 1024 * 1024))
        .build();
}

// ✅ GET with error handling by status code
public Mono<StockResponse> getStock(Long productId) {
    return webClient.get()
        .uri("/api/inventory/{id}/stock", productId)
        .retrieve()
        .onStatus(HttpStatusCode::is4xxClientError, resp ->
            resp.bodyToMono(String.class)
                .map(body -> new ProductNotFoundException("Product " + productId)))
        .onStatus(HttpStatusCode::is5xxServerError, resp ->
            Mono.error(new ServiceUnavailableException("Inventory service down")))
        .bodyToMono(StockResponse.class);
}

// ✅ POST
public Mono<ReservationResponse> reserve(ReserveRequest request) {
    return webClient.post()
        .uri("/api/inventory/reserve")
        .bodyValue(request)
        .retrieve()
        .bodyToMono(ReservationResponse.class);
}

// ✅ Parallel calls with Mono.zip()
public Mono<OrderDetailResponse> getOrderDetail(Long orderId) {
    Mono<OrderResponse> order = orderClient.getOrder(orderId);
    Mono<UserResponse> user = order.flatMap(o -> userClient.getUser(o.userId()));

    return Mono.zip(order, user)
        .map(tuple -> new OrderDetailResponse(tuple.getT1(), tuple.getT2()));
}
```

---

## Server-Sent Events (SSE) / Streaming

```java
// ✅ SSE endpoint
@GetMapping(value = "/events/orders", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<OrderEvent>> streamOrderEvents() {
    return orderEventService.getEventStream()
        .map(event -> ServerSentEvent.<OrderEvent>builder()
            .id(String.valueOf(event.getId()))
            .event("order-update")
            .data(event)
            .build());
}

// ✅ Streaming JSON (ndjson) — each item flushed immediately
@GetMapping(value = "/products/stream", produces = MediaType.APPLICATION_NDJSON_VALUE)
public Flux<ProductResponse> streamProducts() {
    return productService.findAll();
}
```

> When testing SSE with `StepVerifier` or `WebTestClient`, always use `thenCancel()` for infinite streams — without it the test waits forever.

---

## Reactive Security (SecurityWebFilterChain)

WebFlux uses `SecurityWebFilterChain`, **not** `SecurityFilterChain`.

```java
@Configuration
@EnableWebFluxSecurity
public class SecurityConfig {

    @Bean
    public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http) {
        return http
            .csrf(csrf -> csrf.disable())
            .authorizeExchange(auth -> auth
                .pathMatchers("/api/public/**").permitAll()
                .pathMatchers(HttpMethod.GET, "/api/v1/**").hasRole("USER")
                .anyExchange().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .build();
    }
}

// ✅ Accessing authenticated user reactively
@GetMapping("/profile")
public Mono<UserResponse> getProfile(Authentication authentication) {
    return userService.findByEmail(authentication.getName());
}

// ✅ Via ReactiveSecurityContextHolder
public Mono<String> getCurrentUserId() {
    return ReactiveSecurityContextHolder.getContext()
        .map(ctx -> ctx.getAuthentication().getName());
}
```

---

## R2DBC — Reactive Database Access

```java
// ✅ R2DBC repository — reactive replacement for JpaRepository
public interface ProductRepository extends ReactiveCrudRepository<Product, Long> {

    Flux<Product> findByCategoryId(Long categoryId);

    @Query("SELECT * FROM products WHERE price < :maxPrice")
    Flux<Product> findCheaperThan(BigDecimal maxPrice);
}

// ✅ Service with @Transactional (auto-configured with ReactiveTransactionManager)
@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository productRepository;

    @Transactional
    public Mono<Product> create(CreateProductRequest req) {
        return productRepository.save(Product.from(req));
    }

    public Flux<Product> findAll() {
        return productRepository.findAll();
    }
}
```

> R2DBC does **not** support `@OneToMany` / `@ManyToOne` JPA relationships. Model relationships via foreign keys and separate queries, or use `DatabaseClient` for joins.

---

## ❌ Anti-Patterns: Blocking in Reactive Context

### `.block()` on Netty event loop — DEADLOCK

```java
// ❌ DANGEROUS: blocks event loop → deadlock / IllegalStateException
@GetMapping("/{id}")
public Mono<ProductResponse> getById(@PathVariable Long id) {
    ProductResponse p = productService.findById(id).block();  // ❌
    return Mono.just(p);
}

// ✅ Keep the chain reactive
@GetMapping("/{id}")
public Mono<ProductResponse> getById(@PathVariable Long id) {
    return productService.findById(id);
}
```

Spring WebFlux throws `IllegalStateException: block()/blockFirst()/blockLast() are blocking, which is not supported in thread reactor-http-nio-X`. Even if it doesn't throw immediately, it starves the event loop under load.

### Blocking JDBC directly in reactive chain

```java
// ❌ BAD: blocking call on event loop thread
public Mono<Order> createOrder(CreateOrderRequest req) {
    return Mono.just(req)
        .map(r -> jdbcOrderRepository.save(r.toEntity()));  // ❌ blocking!
}

// ✅ Offload to boundedElastic
public Mono<Order> createOrder(CreateOrderRequest req) {
    return Mono.fromCallable(() -> jdbcOrderRepository.save(req.toEntity()))
               .subscribeOn(Schedulers.boundedElastic());
}

// ✅ Best: migrate to R2DBC
public Mono<Order> createOrder(CreateOrderRequest req) {
    return r2dbcOrderRepository.save(req.toEntity());
}
```

### `@Transactional` + R2DBC requires reactive transaction manager

```java
// ❌ Standard @Transactional doesn't work with R2DBC without the right setup
// ✅ spring-boot-starter-data-r2dbc auto-configures ReactiveTransactionManager
// @Transactional then works reactively — ensure the starter is on the classpath
```

### `subscribe()` inside a chain — detached stream

```java
// ❌ BAD: errors swallowed, result detached from main chain
public Mono<Order> processOrder(Order order) {
    auditService.log(order).subscribe();  // ❌
    return orderRepository.save(order);
}

// ✅ Chain it properly
public Mono<Order> processOrder(Order order) {
    return orderRepository.save(order)
        .flatMap(saved -> auditService.log(saved).thenReturn(saved));
}
```
