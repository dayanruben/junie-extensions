# AssertJ + Mockito / BDDMockito Idioms

> **Scope**: This reference covers AssertJ and BDDMockito patterns **in the context of Spring Boot tests** — including `@MockitoBean`, context caching, and Boot 4 migration.
> For non-Spring unit tests (plain JUnit 5 + `@ExtendWith(MockitoExtension.class)`, `@Mock`, `@InjectMocks`), see `test-quality` skill instead.

Quick-reference for assertion and mocking patterns used across all Spring Boot test types. Prefer `org.mockito.BDDMockito` (`given/willReturn`) over classic Mockito (`when/thenReturn`) for readability and BDD alignment.

---

## Static Import Conventions

```java
// AssertJ
import static org.assertj.core.api.Assertions.*;

// BDDMockito (preferred over classic Mockito)
import static org.mockito.BDDMockito.*;

// Hamcrest (for MockMvc jsonPath — not AssertJ)
import static org.hamcrest.Matchers.*;
```

---

## AssertJ — Scalar Assertions

```java
import static org.assertj.core.api.Assertions.*;

assertThat(user.getEmail()).isEqualTo("alice@example.com");
assertThat(user.getAge()).isGreaterThan(18).isLessThan(100);
assertThat(user.getName()).startsWith("Al").endsWith("ce").hasSize(5);
assertThat(user.isActive()).isTrue();
assertThat(user.getDeletedAt()).isNull();
assertThat(optional).isPresent().hasValue("expected");
assertThat(optional).isEmpty();

// BigDecimal — ALWAYS use isEqualByComparingTo (not isEqualTo — scale-aware comparison)
assertThat(price).isEqualByComparingTo(BigDecimal.valueOf(9.99));
assertThat(price).isEqualByComparingTo("9.99");
```

---

## AssertJ — Collection Assertions

```java
// Ordering matters
assertThat(list).containsExactly("alpha", "beta", "gamma");

// Ordering doesn't matter
assertThat(list).containsExactlyInAnyOrder("gamma", "alpha", "beta");

// Subset check
assertThat(list).contains("alpha", "gamma");
assertThat(list).hasSize(3).isNotEmpty();

// Filtering
assertThat(orders)
    .filteredOn(o -> o.getStatus() == OrderStatus.PENDING)
    .hasSize(2);

// Extracting single field
assertThat(users)
    .extracting(User::getEmail)
    .containsExactlyInAnyOrder("alice@example.com", "bob@example.com");

// Multi-field extraction (tuple)
import static org.assertj.core.groups.Tuple.tuple;

assertThat(orders)
    .extracting(Order::getStatus, Order::getTotal)
    .containsExactlyInAnyOrder(
        tuple(OrderStatus.PENDING, new BigDecimal("50.00")),
        tuple(OrderStatus.SHIPPED, new BigDecimal("100.00"))
    );

// allSatisfy / anySatisfy
assertThat(users).allSatisfy(user -> {
    assertThat(user.getEmail()).contains("@");
    assertThat(user.isActive()).isTrue();
});
assertThat(users).anySatisfy(user ->
    assertThat(user.getRoles()).contains("ADMIN"));
```

---

## AssertJ — Exception Assertions

```java
// Preferred pattern
assertThatThrownBy(() -> userService.findById(99L))
    .isInstanceOf(UserNotFoundException.class)
    .hasMessage("User not found: 99")
    .hasMessageContaining("99");

// assertThatExceptionOfType (fluent alternative)
assertThatExceptionOfType(UserNotFoundException.class)
    .isThrownBy(() -> userService.findById(99L))
    .withMessage("User not found: 99");

// catchThrowable — when you need to inspect the exception further
Throwable thrown = catchThrowable(() -> orderService.cancel(1L));
assertThat(thrown)
    .isInstanceOf(IllegalStateException.class)
    .hasMessageContaining("already cancelled");

// assertThatNoException — verify no exception is thrown
assertThatNoException().isThrownBy(() -> validator.validate(validRequest));
```

---

## BDDMockito — Stubbing

```java
import static org.mockito.BDDMockito.*;

// given / willReturn
given(userRepository.findById(1L))
    .willReturn(Optional.of(new User(1L, "alice@example.com")));

// willThrow
given(userRepository.findById(99L))
    .willThrow(new UserNotFoundException(99L));

// willAnswer — dynamic return value
given(idGenerator.nextId())
    .willAnswer(inv -> UUID.randomUUID().toString());

// Void methods
willDoNothing().given(emailService).sendWelcome(any(User.class));
willThrow(new EmailException("SMTP down")).given(emailService).sendWelcome(any(User.class));
```

---

## Argument Matchers

```java
// Exact value (no matcher needed)
given(repo.findById(1L)).willReturn(Optional.of(user));

// any() / any(Class) — type-safe
given(repo.save(any(User.class))).willReturn(savedUser);

// RULE: If ANY argument uses a matcher, ALL arguments must use matchers
// WRONG (mixes matcher with literal):
given(repo.findByEmailAndActive(eq("alice@example.com"), true)).willReturn(Optional.of(user));

// CORRECT:
given(repo.findByEmailAndActive(eq("alice@example.com"), eq(true))).willReturn(Optional.of(user));

// argThat — inline predicate
given(repo.save(argThat(u -> "alice@example.com".equals(u.getEmail()))))
    .willReturn(savedUser);

// Common matchers:
// any()               — any non-null value
// any(Class)          — any instance of class
// anyString()         — any non-null String (null requires isNull())
// anyLong() / anyInt() — any primitive wrapper
// eq(value)           — exact equality
// isNull() / notNull() — null checks
// argThat(predicate)  — custom predicate
```

---

## BDDMockito — Verification

```java
// then(mock).should() replaces Mockito.verify(mock)
then(emailService).should().sendWelcome(any(User.class));       // called once
then(retryService).should(times(3)).attempt(any());             // called N times
then(emailService).should(never()).sendAlert(any());            // never called
then(cache).should(atLeastOnce()).evict(anyString());           // at least once
then(emailService).shouldHaveNoMoreInteractions();              // use sparingly — brittle
```

---

## ArgumentCaptor

Capture the argument passed to a mock method and assert on it:

```java
import org.mockito.ArgumentCaptor;
import org.mockito.Captor;

@Captor
ArgumentCaptor<EmailNotification> captor;

@Test
void registerUser_sendsWelcomeEmail() {
    userService.register(new RegisterRequest("alice@example.com", "Alice"));

    then(emailService).should().send(captor.capture());

    EmailNotification notification = captor.getValue();
    assertThat(notification.getTo()).isEqualTo("alice@example.com");
    assertThat(notification.getSubject()).contains("Welcome");
}

// Multiple invocations
then(auditLog).should(times(2)).record(captor.capture());
List<AuditEvent> events = captor.getAllValues();
assertThat(events).extracting(AuditEvent::getAction)
    .containsExactly("USER_CREATED", "EMAIL_SENT");
```

`@Captor` requires `@ExtendWith(MockitoExtension.class)` or Spring's test support to inject. Alternatively: `ArgumentCaptor<EmailNotification> captor = ArgumentCaptor.forClass(EmailNotification.class);`

---

## @MockitoBean vs @MockitoSpyBean (Boot 4)

Version note: this section is specific to Boot 4 style APIs. If the project is on Boot 3.x, keep using `@MockBean` / `@SpyBean`.

```java
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.context.bean.override.mockito.MockitoSpyBean;

// @MockitoBean — full mock, all methods return defaults unless stubbed
// Replaces @MockBean from org.springframework.boot.test.mock.mockito in Boot 4
@MockitoBean
OrderService orderService;

// @MockitoSpyBean — wraps the REAL bean, calls real implementation unless stubbed
// Replaces @SpyBean in Boot 4
@MockitoSpyBean
AuditService auditService;

// With spy, only override what you need:
willDoNothing().given(auditService).record(any(AuditEvent.class));
// All other AuditService methods run the real code
```

**Warning on `@MockitoSpyBean`**: The real bean must be loadable in the test slice context. In `@WebMvcTest`, only web-layer beans load — `@MockitoSpyBean` on a service will fail if all of that service's dependencies aren't satisfied.

---

## Strict Stubbing and UnnecessaryStubbingException

Mockito 2+ uses strict stubbing by default (via `MockitoExtension`): if you define a stub that is never called in the test, Mockito throws `UnnecessaryStubbingException`.

```java
// CAUSE: stub defined in @BeforeEach but only one test uses it
@BeforeEach
void setUp() {
    given(repo.findById(1L)).willReturn(Optional.of(user)); // UnnecessaryStubbingException
}

@Test
void testA() {
    repo.findById(1L); // only testA uses this stub
}

@Test
void testB() {
    // testB doesn't call findById — UnnecessaryStubbingException for testB!
}
```

**Fixes**:

```java
// Fix A (preferred): Move stubs into each test that needs them
@Test
void testA() {
    given(repo.findById(1L)).willReturn(Optional.of(user));
    repo.findById(1L);
}

// Fix B: Use lenient() for shared setup stubs
@BeforeEach
void setUp() {
    lenient().when(repo.findById(anyLong())).thenReturn(Optional.of(user));
}

// Fix C (avoid): Relax at class level
@MockitoSettings(strictness = Strictness.LENIENT)
class MyServiceTest { }
```

---

## MockMvc + Hamcrest vs AssertJ

`MockMvc.andExpect()` uses **Hamcrest** matchers (not AssertJ) in `jsonPath`:

```java
import static org.hamcrest.Matchers.*;

.andExpect(jsonPath("$.items", hasSize(3)))
.andExpect(jsonPath("$.name", containsString("Widget")))
.andExpect(jsonPath("$.price", greaterThan(0.0)))
.andExpect(jsonPath("$.tags", hasItem("featured")))
.andExpect(jsonPath("$.ids", containsInAnyOrder(1, 2, 3)))
```

For complex body assertions, extract and use AssertJ:

```java
String json = mockMvc.perform(get("/orders/1"))
    .andExpect(status().isOk())
    .andReturn().getResponse().getContentAsString();
OrderDto order = objectMapper.readValue(json, OrderDto.class);
assertThat(order.getItems()).hasSize(3);
```

---

## Common Mistakes

```java
// MISTAKE 1: Stubbing after the act — always stub BEFORE the method call
orderService.processOrder(request);       // act (wrong order)
given(repo.findById(1L)).willReturn(...); // WRONG — stub too late

// MISTAKE 2: Mixing matchers with literals
given(repo.find(eq("alice"), true)).willReturn(...);      // ERROR
given(repo.find(eq("alice"), eq(true))).willReturn(...);  // CORRECT

// MISTAKE 3: Over-verifying implementation details
verify(repo).findById(1L);
verify(mapper).toDto(any()); // noise — tests implementation, not behavior

// CORRECT: verify observable side effects only
then(emailService).should().sendConfirmation(any(Order.class));

// MISTAKE 4: Mocking DTOs or domain value objects
// Don't mock simple data carriers — use real instances instead

// MISTAKE 5: Defined stubs in @BeforeEach but not all tests use them
// Move stubs into the tests that actually need them
```

---

## Boot 3.x → 4.x Mockito/AssertJ Changes

Use this table only when the project is actively migrating to Boot 4.x.

| Area | Boot 3.x | Boot 4.x |
|---|---|---|
| `@MockBean` | `org.springframework.boot.test.mock.mockito.MockBean` | Removed — use `@MockitoBean` from `org.springframework.test.context.bean.override.mockito` |
| `@SpyBean` | `org.springframework.boot.test.mock.mockito.SpyBean` | Removed — use `@MockitoSpyBean` |
| `MockMvc` (primary) | `MockMvc` | `MockMvc` or `RestTestClient` via `@AutoConfigureRestTestClient` |
| AssertJ | 3.24.x | 3.26.x+ |
| Strict stubbing | Default via `MockitoExtension` | Same |

---

## Quick Reference

| Need | Use |
|---|---|
| Stub return value | `given(mock.method(args)).willReturn(value)` |
| Stub void to do nothing | `willDoNothing().given(mock).method(args)` |
| Stub to throw | `given(mock.method(args)).willThrow(ex)` |
| Verify call happened | `then(mock).should().method(args)` |
| Verify never called | `then(mock).should(never()).method(any())` |
| Capture argument | `then(mock).should().method(captor.capture()); captor.getValue()` |
| Assert exception | `assertThatThrownBy(() -> ...).isInstanceOf(X.class)` |
| Assert collection fields | `assertThat(list).extracting(X::getField).contains(...)` |
| Assert BigDecimal | `assertThat(val).isEqualByComparingTo(BigDecimal.valueOf(x))` |
| Hamcrest in MockMvc jsonPath | `jsonPath("$.field", is("value"))` |


---

# Cross-Cutting Testing Patterns

Strategic guidance that applies across all Spring test types: the testing pyramid, context caching, behavior-over-implementation, Boot 4 readiness, and universal anti-patterns.

Source: Andy Wilkinson's "Testing Spring Boot Applications" (SpringOne 2019), Spring Boot team guidance.

---

## Testing Pyramid — Slice Selection

Always use the narrowest slice that covers your test objective. Only escalate when you're testing cross-cutting behavior (security + service + persistence together).

| Priority | Annotation | Scope | Typical Startup |
|---|---|---|---|
| 1 | Unit test (no Spring) | Single class, no context | < 100ms |
| 2 | `@WebMvcTest` | MVC controllers only | 1–3s |
| 3 | `@DataJpaTest` | JPA entities + repositories only | 2–5s |
| 4 | `@WebFluxTest` | Reactive controllers only | 1–3s |
| 5 | `@SpringBootTest` | Full context | 5–30s |
| 6 | `@SpringBootTest(RANDOM_PORT)` | Full context + real server | 10–30s+ |

**Decision rules**:
- `@WebMvcTest` → any REST controller test (security filter chain auto-configured)
- `@DataJpaTest` → custom JPQL/SQL queries, projections, auditing, constraints
- `@WebFluxTest` → reactive controller tests
- `@SpringBootTest` → cross-layer integration: service + security + persistence
- `@SpringBootTest(RANDOM_PORT)` → WebSocket (requires real TCP), or HTTP client integration tests

---

## Context Caching — The #1 Performance Pitfall

Spring Test caches `ApplicationContext` instances across test classes. The cache is keyed by the exact combination of configuration. Any difference creates a new context.

**Context cache killers**:

```java
// Each unique @MockitoBean combination creates a new context cache entry

// Context A (class 1):
@WebMvcTest(UserController.class)
class UserControllerTest {
    @MockitoBean UserService userService;
}

// Context B (class 2 — different mock set):
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @MockitoBean OrderService orderService;
    @MockitoBean UserService userService;   // same mock, different combination → new context
}
```

**Other cache invalidators**:
- `@DirtiesContext` — evicts the context after the test class (use only when test genuinely corrupts state)
- `@ActiveProfiles` — different profiles = different contexts
- `@Import` with different `@TestConfiguration` classes = different contexts
- `@TestPropertySource` with different properties = different contexts

**Best practice**: Standardize your mock sets. If multiple test classes all test controllers that depend on `UserService`, make them use the same `@MockitoBean` set so they share a context.

Default cache limit: 32 contexts. Beyond that, LRU eviction + recreation causes catastrophic slowdown in large test suites.

---

## @DirtiesContext — Use Sparingly

`org.springframework.test.annotation.DirtiesContext` (`@DirtiesContext`) evicts the application context after the annotated test class or method. It forces a full context rebuild for subsequent tests.

When to use:
- Test modifies `@Configuration` beans or shared static state
- Test starts background threads that pollute the context

When NOT to use:
- Test fails and you want a fresh context — fix the test instead
- Routine cleanup — use `@Transactional` rollback or `@BeforeEach` cleanup

---

## Behavior > Implementation

Across all domains, assert **observable outcomes**, not implementation details.

| Domain | Assert This (Behavior) | Not This (Implementation) |
|---|---|---|
| MVC | HTTP status code + response body fields | `verify(service).findById(1L)` |
| JPA | Re-fetched entity state after flush+clear | In-memory entity state before flush |
| Security | 401/403 status codes for unauthorized requests | Internal filter chain invocations |
| WebFlux | StepVerifier signals + emitted values | `.block()` result |
| WebSocket | Message received in `BlockingQueue` | Internal broker routing |
| Service | Return value, exception type, side-effect (email sent) | Number of repository calls |

---

## Universal Anti-Patterns

| Anti-Pattern | Why It's Bad | Fix |
|---|---|---|
| `@SpringBootTest` for every test | Slow startup, hides what you're testing | Use narrowest slice |
| Verifying mock call counts instead of behavior | Brittle — couples to implementation, breaks on refactor | Assert observable outcomes |
| String-matching JSON responses (`content().string(...)`) | Breaks on field ordering and whitespace | Use `jsonPath()` |
| Disabling security in tests (`addFilters = false`) | False confidence — you're not testing what runs in production | Test security explicitly |
| Using H2 for dialect-sensitive queries | H2 ≠ PostgreSQL — tests pass locally, fail in production | Use Testcontainers with production DB |
| `.block()` in reactive tests | Hides reactive errors, breaks scheduler assumptions | Use `StepVerifier` or `WebTestClient` |
| `Thread.sleep()` in async tests | Flaky, non-deterministic, slow | Use `Awaitility`, `BlockingQueue.poll(timeout)`, or `CountDownLatch` |
| Over-mocking value objects and DTOs | Maintenance burden, no value | Use real instances |
| `@DirtiesContext` to paper over test failures | Hides the real problem | Fix the test isolation issue instead |
| Unused stubs in `@BeforeEach` | `UnnecessaryStubbingException`, signal-to-noise ratio | Move stubs into the tests that use them |

---

## Awaitility for Async Assertions

For asynchronous operations that don't fit `BlockingQueue` or `StepVerifier`:

```java
import org.awaitility.Awaitility;
import static org.awaitility.Awaitility.*;

// Poll until condition is true, with timeout
await()
    .atMost(Duration.ofSeconds(10))
    .pollInterval(Duration.ofMillis(100))
    .untilAsserted(() -> {
        assertThat(emailSentCount.get()).isEqualTo(1);
    });

// Simple predicate poll
await()
    .atMost(Duration.ofSeconds(5))
    .until(() -> auditLog.contains("ORDER_CREATED"));
```

Awaitility is far more reliable than `Thread.sleep()` and gives clear timeout messages when the condition isn't met.

---

## Boot 4 Readiness Checklist

### Stack Requirements
- Java 21+
- Jakarta namespaces only (no `javax.*`)
- No deprecated Boot 2/3 APIs

### Annotation Migration

| Boot 3.x | Boot 4.x | New Package |
|---|---|---|
| `@MockBean` | `@MockitoBean` | `org.springframework.test.context.bean.override.mockito` |
| `@SpyBean` | `@MockitoSpyBean` | `org.springframework.test.context.bean.override.mockito` |
| `@WebMvcTest` import | Same name, new package | `org.springframework.boot.webmvc.test.autoconfigure` |
| `@DataJpaTest` import | Same name, new package | `org.springframework.boot.data.jpa.test.autoconfigure` |
| `TestEntityManager` import | Same name, new package | `org.springframework.boot.jpa.test.autoconfigure` |
| `@WebFluxTest` import | Same name, new package | `org.springframework.boot.webflux.test.autoconfigure` |

### Quick Migration Script

```bash
# Find all imports that need updating
grep -r "org.springframework.boot.test.autoconfigure.orm.jpa" src/test/
grep -r "org.springframework.boot.test.autoconfigure.web.servlet" src/test/
grep -r "org.springframework.boot.test.autoconfigure.web.reactive" src/test/
grep -r "org.springframework.boot.test.mock.mockito.MockBean" src/test/
grep -r "org.springframework.boot.test.mock.mockito.SpyBean" src/test/
```

### AOT Safety Rules

Spring Boot 4 supports AOT compilation (GraalVM native image). Test code that uses reflection hacks or classloader manipulation may not work with AOT.

- Prefer constructor injection over field injection
- Prefer explicit bean wiring over `@Autowired` field injection in test configs
- Prefer framework-provided test annotations over custom classloader manipulation
- Avoid `ReflectionTestUtils` for production code fields — fix the visibility instead

---

## Context Caching Optimization Patterns

### Pattern 1: Shared Base Test Class

```java
// Base class that all @WebMvcTest tests extend — same mock set = shared context
@WebMvcTest
abstract class AbstractWebMvcTest {
    @MockitoBean UserService userService;
    @MockitoBean OrderService orderService;
    @MockitoBean ProductService productService;
}

// Subclass narrows to specific controller — no new mocks → shares context
@WebMvcTest(UserController.class)
class UserControllerTest extends AbstractWebMvcTest { ... }

@WebMvcTest(OrderController.class)
class OrderControllerTest extends AbstractWebMvcTest { ... }
```

Trade-off: base class includes mocks that specific tests don't need. Acceptable for small suites; may not scale.

### Pattern 2: @TestConfiguration Shared Config

```java
// Shared test infrastructure in one place
@TestConfiguration
public class SharedTestConfig {
    @Bean @Primary
    public Clock testClock() { return Clock.fixed(Instant.parse("2026-01-01T00:00:00Z"), ZoneOffset.UTC); }
}

// Import everywhere — same config class = same cache key contribution
@WebMvcTest(UserController.class)
@Import(SharedTestConfig.class)
class UserControllerTest { ... }
```

---

## Test Naming Conventions

Consistent naming makes failures readable:

```
methodUnderTest_scenarioDescription_expectedBehavior

findById_withValidId_returnsUser()
findById_withUnknownId_throwsNotFoundException()
createOrder_whenStockInsufficient_throwsOutOfStockException()
getOrder_withoutAuth_returns401()
getOrder_asUser_withDifferentOwner_returns403()
```

The pattern: what you're testing (`findById`), the condition (`withValidId`), the expectation (`returnsUser`).
