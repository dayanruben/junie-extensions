# JPA Testing: Testcontainers and @DataJpaTest vs @SpringBootTest

Covers the `@ServiceConnection` pattern for Testcontainers, why H2 is insufficient for production-like testing, and when to choose `org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest` (`@DataJpaTest`) versus `org.springframework.boot.test.context.SpringBootTest` (`@SpringBootTest`).

---

## Why Not H2

> "You should use the same database engine that you also run in production to validate the very same behavior that the users are going to experience."
> — Vlad Mihalcea

Problems with H2 as a test database:

- Cannot replicate PostgreSQL-specific features: window functions, `jsonb`, array types, advisory locks
- SQL syntax differences — tests pass on H2 but fail on PostgreSQL with native queries
- Flyway migrations often use PostgreSQL-specific DDL and fail silently on H2
- Sequence generation behavior differs between engines

**Recommendation**: Use Testcontainers with the same database image as production.

**Sources**:
- [Vlad Mihalcea — Testcontainers Database Integration Testing](https://vladmihalcea.com/testcontainers-database-integration-testing/)
- [Testcontainers — Replace H2 with a Real Database](https://testcontainers.com/guides/replace-h2-with-real-database-for-testing/)

---

## @DataJpaTest vs @SpringBootTest: When to Use Which

### What @DataJpaTest Does

`org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest` (`@DataJpaTest`) creates a slice of the Spring application context containing only JPA-related beans:

- Repositories, `EntityManager`, `DataSource`, `JdbcTemplate`
- Auto-configures `org.springframework.boot.jpa.test.autoconfigure.TestEntityManager` (`TestEntityManager`)
- Does NOT load `@Service`, `@Controller`, or non-JPA beans
- Wraps every test method in `@Transactional` with automatic rollback
- Validates entity mappings and JPQL query syntax at startup

By default, `@DataJpaTest` replaces the datasource with an embedded H2 database. Use `@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)` (from `org.springframework.boot.data.jpa.test.autoconfigure.AutoConfigureTestDatabase`) to disable this replacement when using Testcontainers.

### What @SpringBootTest Does

`org.springframework.boot.test.context.SpringBootTest` (`@SpringBootTest`) loads the full application context. It does NOT add `@Transactional` by default — transactions commit normally unless you explicitly add `@Transactional` to the test class.

### Decision Table

| Scenario | Recommendation |
|----------|---------------|
| Testing custom JPQL/native queries in isolation | `@DataJpaTest` with Testcontainers |
| Testing repository + service transaction boundaries | `@SpringBootTest` |
| Testing Flyway migrations + entity mappings | `@DataJpaTest` with real DB |
| Testing DDD aggregate persistence through service layer | `@SpringBootTest` with Testcontainers |
| Project uses PostgreSQL-specific SQL | Must use Testcontainers (not H2) |
| Verifying lazy loading fails outside a transaction | `@SpringBootTest` (no auto-`@Transactional`) |

### Vlad Mihalcea's Case Against @DataJpaTest

Vlad Mihalcea recommends caution with `@DataJpaTest` for integration-style tests:

1. **Automatic `@Transactional` wrapper creates unrealistic conditions**: Service-layer transactions manage production boundaries. Test-level transactions change flush timing and dirty-checking behavior.
2. **Flush behavior mismatch**: `FlushModeType.AUTO` triggers flush before commit. In a rolled-back test transaction, SQL statements may never be sent to the database — constraint violations go undetected.
3. **Masks `LazyInitializationException`**: The test transaction keeps the `EntityManager` open for the entire test, allowing lazy collections to load that would fail in production.

Use `@DataJpaTest` for what it's designed for: isolated query correctness. Use `@SpringBootTest` when verifying production transaction behavior.

**Sources**:
- [Vlad Mihalcea — The best way to clean up test data](https://vladmihalcea.com/clean-up-test-data-spring/)
- [Arho Huttunen — Testing the Persistence Layer with @DataJpaTest](https://www.arhohuttunen.com/spring-boot-datajpatest/)
- [rieckpil — Spring Data JPA Persistence Layer Tests with @DataJpaTest](https://rieckpil.de/test-your-spring-boot-jpa-persistence-layer-with-datajpatest/)

---

## Modern Pattern: @ServiceConnection (Spring Boot 3.1+)

Since Spring Boot 3.1, the recommended approach uses `@TestConfiguration` with `@Bean @ServiceConnection`. No `@DynamicPropertySource` needed.

```java
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.context.annotation.Bean;
import org.testcontainers.containers.PostgreSQLContainer;

@TestConfiguration(proxyBeanMethods = false)
public class TestcontainersConfiguration {

    @Bean
    @ServiceConnection
    PostgreSQLContainer<?> postgresContainer() {
        return new PostgreSQLContainer<>("postgres:16-alpine")
                .withReuse(true);
    }
}
```

`@ServiceConnection` auto-configures `spring.datasource.url`, `spring.datasource.username`, and `spring.datasource.password` from the running container — no manual property wiring.

### Usage with @SpringBootTest

```java
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Import;

@SpringBootTest
@Import(TestcontainersConfiguration.class)
class MyIntegrationTest {
    // Full application context, Testcontainers datasource
}
```

### Usage with @DataJpaTest (requires replace=NONE)

```java
import org.springframework.boot.data.jpa.test.autoconfigure.AutoConfigureTestDatabase;
import org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest;
import org.springframework.context.annotation.Import;

@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Import(TestcontainersConfiguration.class)
class MyRepositoryTest {
    // JPA slice only, Testcontainers datasource
}
```

**Critical**: `@AutoConfigureTestDatabase(replace = NONE)` is required with `@DataJpaTest` when using Testcontainers. Without it, Spring replaces your Testcontainers `DataSource` with H2.

### Container Lifecycle

- Containers declared as `static` beans in `@TestConfiguration` are started once per JVM and shared across all tests in the suite — this is the recommended pattern for CI speed
- `withReuse(true)` keeps containers alive between test runs for faster local development (requires `testcontainers.reuse.enable=true` in `~/.testcontainers.properties`)
- Static containers started this way are automatically stopped by the Testcontainers `ResourceReaper` on JVM shutdown

### Older Pattern: @DynamicPropertySource (still valid)

Before `@ServiceConnection`, the standard approach used `@DynamicPropertySource`:

```java
import org.springframework.boot.data.jpa.test.autoconfigure.AutoConfigureTestDatabase;
import org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class OrderRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres =
            new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}
```

Prefer `@ServiceConnection` for new code — less boilerplate and works with Spring context caching.

---

## Separate @TestConfiguration for Slice vs Full Tests

Wim Deblauwe recommends creating separate `@TestConfiguration` classes:

- **Database-only configuration** for `@DataJpaTest` — starts only the database container
- **Full configuration** for `@SpringBootTest` — starts database plus any other containers (Redis, Kafka, etc.)

This avoids starting unnecessary containers for slice tests:

```java
// For @DataJpaTest only
@TestConfiguration(proxyBeanMethods = false)
public class DatabaseTestConfiguration {
    @Bean
    @ServiceConnection
    PostgreSQLContainer<?> postgresContainer() {
        return new PostgreSQLContainer<>("postgres:16-alpine");
    }
}

// For @SpringBootTest with full infrastructure
@TestConfiguration(proxyBeanMethods = false)
public class FullInfrastructureTestConfiguration {
    @Bean
    @ServiceConnection
    PostgreSQLContainer<?> postgresContainer() {
        return new PostgreSQLContainer<>("postgres:16-alpine");
    }

    @Bean
    @ServiceConnection
    RedisContainer redisContainer() {
        return new RedisContainer("redis:7-alpine");
    }
}
```

**Source**: [Wim Deblauwe — How I Test Production-Ready Spring Boot Applications](https://www.wimdeblauwe.com/blog/2025/07/30/how-i-test-production-ready-spring-boot-applications/)

---

## Context Caching with Testcontainers

Spring caches application contexts across tests. Two tests that import the same `@TestConfiguration` (e.g., `TestcontainersConfiguration`) will reuse the same container and context — this is why static beans are key.

Avoid `@DirtiesContext` unless absolutely necessary — it discards the cached context and forces a full restart. Use it only when your test genuinely modifies shared infrastructure state.

**Sources**:
- [JetBrains Blog — Testing Spring Boot Applications Using Testcontainers](https://blog.jetbrains.com/idea/2024/12/testing-spring-boot-applications-using-testcontainers/)
- [Spring Boot Issue #35121 — Document @ServiceConnection for @DataJpaTest](https://github.com/spring-projects/spring-boot/issues/35121)


---

# JPA Testing: @DataJpaTest Patterns, @Transactional Traps, and Test Data

Core patterns for writing `org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest` (`@DataJpaTest`) tests using `org.springframework.boot.jpa.test.autoconfigure.TestEntityManager` (`TestEntityManager`). Covers flush/clear cycles, the `@Transactional` trap, test data setup, `@Modifying` bulk updates, and what not to test.

---

## The Flush+Clear Pattern

The most common mistake in `@DataJpaTest` tests: saving data and reading it back without flushing and clearing.

```java
import jakarta.persistence.EntityManager;
import org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest;
import org.springframework.boot.jpa.test.autoconfigure.TestEntityManager;
import org.springframework.beans.factory.annotation.Autowired;

@DataJpaTest
class OrderRepositoryTest {

    @Autowired TestEntityManager em;
    @Autowired OrderRepository orderRepository;

    @Test
    void findByCustomerId_returnsMatchingOrders() {
        Customer customer = em.persist(new Customer("alice@example.com"));
        em.persist(new Order(customer, OrderStatus.PENDING));
        em.persist(new Order(customer, OrderStatus.SHIPPED));
        em.flush();   // Send SQL to the database
        em.clear();   // Evict L1 cache — forces repository read to hit the DB

        List<Order> orders = orderRepository.findByCustomerId(customer.getId());

        assertThat(orders).hasSize(2);
    }
}
```

**Why this matters**: Without `flush()`, SQL may never reach the database. Without `clear()`, `findByCustomerId` returns the entity from the first-level cache and never hits the database — your query is never exercised.

Use `persistAndFlush()` as a shorthand when you only need the flush:

```java
User saved = em.persistAndFlush(new User("alice@example.com"));
em.clear(); // Still need clear() to evict from L1 cache
```

---

## The @Transactional-in-Tests Trap

Multiple authoritative sources warn against wrapping integration tests in `@Transactional`:

### Why It Causes Problems

1. **Lazy loading works in tests, fails in production**: The test transaction keeps the `EntityManager` open, allowing lazy collections to load — masking `LazyInitializationException` that production code would hit.

2. **Hibernate may not flush**: Since the transaction is rolled back, `FlushModeType.AUTO` may never trigger. Constraint violations and unique index violations go undetected.

3. **Auto-dirty-checking false positives**: Entities modified within a `@Transactional` test are automatically persisted at flush time without calling `save()`. This would not happen in production if the service layer doesn't explicitly save.

4. **Different transaction boundaries**: Production commits at service method boundaries. Test-level `@Transactional` merges everything into one transaction, hiding multi-transaction bugs.

### When @Transactional IS Appropriate in Tests

- `@DataJpaTest` for isolated query testing (rollback is convenient, transactions are scoped)
- Tests with explicit `flush()` calls — you force SQL execution and detect constraint violations
- Quick verification of derived query methods where production transaction behavior is not the concern

### When to AVOID @Transactional in Tests

- Full integration tests verifying service-layer transaction boundaries
- Tests for `@Modifying` bulk update queries
- Tests that verify lazy loading behavior (see `references/lazy-loading-tests.md`)
- Tests for optimistic locking (`@Version`)

**Sources**:
- [Nurkiewicz — Spring Pitfalls: Transactional Tests Considered Harmful](https://nurkiewicz.com/2011/11/spring-pitfalls-transactional-tests.html)
- [rieckpil — Spring Boot Testing Pitfall: Transaction Rollback](https://rieckpil.de/spring-boot-testing-pitfall-transaction-rollback-in-tests/)
- [Marco Behler — Should My Tests Be @Transactional?](https://www.marcobehler.com/2014/06/25/should-my-tests-be-transactional)

---

## Core Test Patterns

### Testing Derived Query Methods

```java
@Test
void findByEmailAndActiveTrue_returnsOnlyActiveUsers() {
    em.persist(new User("bob@example.com", true));
    em.persist(new User("bob@example.com", false));
    em.flush();
    em.clear();

    List<User> results = userRepository.findByEmailAndActiveTrue("bob@example.com");

    assertThat(results).hasSize(1);
    assertThat(results.get(0).isActive()).isTrue();
}

@Test
void findTop3ByOrderByCreatedAtDesc_returnsLatestThree() {
    for (int i = 0; i < 5; i++) {
        em.persist(new Product("P" + i, BigDecimal.TEN));
    }
    em.flush();
    em.clear();

    List<Product> top3 = productRepository.findTop3ByOrderByCreatedAtDesc();
    assertThat(top3).hasSize(3);
}
```

### Testing Custom @Query Methods

```java
// Repository method:
// @Query("SELECT o FROM Order o WHERE o.status = :status AND o.total >= :minTotal")
// List<Order> findByStatusAndMinTotal(@Param("status") OrderStatus status,
//                                     @Param("minTotal") BigDecimal minTotal);

@Test
void findByStatusAndMinTotal_filtersCorrectly() {
    Customer c = em.persist(new Customer("alice@example.com"));
    em.persist(orderWith(c, OrderStatus.PENDING, new BigDecimal("50.00")));
    em.persist(orderWith(c, OrderStatus.PENDING, new BigDecimal("150.00")));
    em.persist(orderWith(c, OrderStatus.SHIPPED, new BigDecimal("200.00")));
    em.flush();
    em.clear();

    List<Order> results = orderRepository
        .findByStatusAndMinTotal(OrderStatus.PENDING, new BigDecimal("100.00"));

    assertThat(results).hasSize(1);
    assertThat(results.get(0).getTotal()).isEqualByComparingTo("150.00");
}
```

### Testing Pagination

```java
@Test
void findAll_paginated_returnsCorrectPage() {
    for (int i = 0; i < 10; i++) {
        em.persist(new Product("P" + i, BigDecimal.TEN));
    }
    em.flush();
    em.clear();

    Pageable pageable = PageRequest.of(0, 3, Sort.by("name").ascending());
    Page<Product> page = productRepository.findAll(pageable);

    assertThat(page.getContent()).hasSize(3);
    assertThat(page.getTotalElements()).isEqualTo(10);
    assertThat(page.getTotalPages()).isEqualTo(4);
    assertThat(page.getContent().get(0).getName()).isEqualTo("P0");
}
```

### Testing Specifications

```java
@Test
void spec_filtersByCategory() {
    em.persist(new Product("Widget", "ELECTRONICS", BigDecimal.TEN));
    em.persist(new Product("Screw", "HARDWARE", BigDecimal.ONE));
    em.flush();
    em.clear();

    List<Product> results = productRepository
        .findAll(ProductSpecs.hasCategory("ELECTRONICS"));

    assertThat(results).hasSize(1);
    assertThat(results.get(0).getName()).isEqualTo("Widget");
}
```

### Testing Interface Projections

```java
// interface UserSummary { String getEmail(); String getDisplayName(); }
// List<UserSummary> findByActiveTrue();

@Test
void findByActiveTrue_returnsProjection() {
    em.persist(new User("alice@example.com", "Alice Smith", true));
    em.persist(new User("bob@example.com", "Bob Jones", false));
    em.flush();
    em.clear();

    List<UserSummary> summaries = userRepository.findByActiveTrue();

    assertThat(summaries).hasSize(1);
    assertThat(summaries.get(0).getEmail()).isEqualTo("alice@example.com");
    assertThat(summaries.get(0).getDisplayName()).isEqualTo("Alice Smith");
}
```

### Testing Entity Auditing

```java
import org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.context.annotation.Import;
import org.springframework.data.jpa.repository.config.EnableJpaAuditing;

@DataJpaTest
@Import(AuditedEntityTest.AuditingConfig.class)
class AuditedEntityTest {

    @TestConfiguration
    @EnableJpaAuditing
    static class AuditingConfig {}

    @Autowired TestEntityManager em;
    @Autowired ProductRepository productRepository;

    @Test
    void save_populatesAuditDates() {
        Product p = productRepository.save(new Product("Widget", BigDecimal.TEN));
        em.flush();
        em.clear();

        Product loaded = productRepository.findById(p.getId()).orElseThrow();
        assertThat(loaded.getCreatedAt()).isNotNull();
        assertThat(loaded.getUpdatedAt()).isNotNull();
    }
}
```

**Note**: `@EnableJpaAuditing` is not loaded by `@DataJpaTest` by default — you must import a `@TestConfiguration` that enables it.

### Constraint Violation Testing

```java
@Test
void save_duplicateEmail_throwsConstraintViolation() {
    em.persist(new User("alice@example.com"));
    em.flush();

    assertThatThrownBy(() -> {
        em.persist(new User("alice@example.com"));
        em.flush(); // flush() triggers the database constraint check
    }).isInstanceOf(DataIntegrityViolationException.class);
}
```

**Key**: The second `em.flush()` is required — without it, Hibernate may batch the insert and you may not see the violation.

### Transaction Rollback Control

```java
// @DataJpaTest wraps each test in @Transactional → rollback after each test.
// Override when you need a real commit:

@Test
@Commit   // Commits the transaction — use sparingly, clean up manually
void saveOrder_commitsSuccessfully() { /* ... */ }

// Or disable the wrapping transaction entirely:
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@DataJpaTest
@Transactional(propagation = Propagation.NOT_SUPPORTED)
class OrderRepositoryCommitTest {
    // Must clean up test data manually in @BeforeEach
    @BeforeEach
    void cleanUp() {
        orderRepository.deleteAll();
    }
}
```

---

## Test Data Setup and Cleanup

### Vlad Mihalcea's Approach: Truncate Before Each Test

> "Execute cleanup **before** each test, not after. Cleanup in `@AfterEach` may not run during test failures or debugging sessions."

Since Hibernate 6.2, use the `SchemaManager` API:

```java
import org.hibernate.engine.spi.SessionFactoryImplementor;

@Autowired EntityManagerFactory entityManagerFactory;

@BeforeEach
void cleanUp() {
    entityManagerFactory
        .unwrap(SessionFactoryImplementor.class)
        .getSchemaManager()
        .truncateMappedObjects();
}
```

Benefits: truncates all tables mapped to JPA entities, respects foreign key order automatically, works with any database.

**Source**: [Vlad Mihalcea — The best way to clean up test data](https://vladmihalcea.com/clean-up-test-data-spring/)

### @Sql for Complex Data Setup

For complex test data, use `@Sql` to load SQL scripts:

```java
import org.springframework.test.context.jdbc.Sql;

@Test
@Sql("/test-data/users.sql")
void complexQuery_shouldReturnExpectedResults() { ... }
```

Caveat: SQL scripts need maintenance when schemas change. Prefer programmatic setup for resilience.

### Do NOT Use the Repository Under Test for Setup

```java
// WRONG: circular logic — repository bug masks both setup and assertion failure
userRepository.save(new User("alice@example.com"));
List<User> users = userRepository.findByActiveTrue();

// CORRECT: use TestEntityManager for setup
em.persistAndFlush(new User("alice@example.com", true));
em.clear();
List<User> users = userRepository.findByActiveTrue();
```

Using the same repository for both setup and assertion creates circular logic that masks bugs in the method under test.

**Source**: [Arho Huttunen — Testing the Persistence Layer](https://www.arhohuttunen.com/spring-boot-datajpatest/)

---

## Testing @Modifying Bulk Updates

### The Pattern

```java
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.transaction.annotation.Transactional;

@Modifying
@Transactional  // REQUIRED when extending Repository (not JpaRepository)
@Query("UPDATE User u SET u.status = :status WHERE u.id = :id")
int updateStatus(@Param("id") Long id, @Param("status") String status);
```

When extending `org.springframework.data.repository.Repository` (not `JpaRepository`), `@Modifying` queries do NOT inherit `@Transactional`. Without it, you get:

```
InvalidDataAccessApiUsageException: Executing an update/delete query
```

### Testing Bulk Updates

```java
@Test
void updateStatus_shouldModifyEntity() {
    User user = em.persistAndFlush(new User("Alice", "ACTIVE"));
    em.clear();

    int updated = userRepository.updateStatus(user.getId(), "INACTIVE");

    assertThat(updated).isEqualTo(1);
    em.clear(); // Clear stale cached state after bulk update
    User reloaded = em.find(User.class, user.getId());
    assertThat(reloaded.getStatus()).isEqualTo("INACTIVE");
}
```

Steps:
1. Insert test data via `TestEntityManager`
2. Execute the bulk update
3. `em.clear()` — bulk updates bypass Hibernate dirty-checking; the L1 cache holds stale state
4. Re-read and assert

### clearAutomatically for Safety

`@Modifying(clearAutomatically = true)` clears the persistence context automatically after the bulk update:

```java
@Modifying(clearAutomatically = true)
@Transactional
@Query("UPDATE StepExecution s SET s.itemCount = :count, s.version = s.version + 1 WHERE s.id = :id")
int updateItemCount(@Param("id") UUID id, @Param("count") long count);
```

### Testing Version Increment in Bulk Updates

JPQL bulk updates bypass Hibernate's dirty-checking and optimistic lock version increment. If you manage the version field manually in the query, verify it:

```java
@Test
void bulkUpdate_shouldIncrementVersion() {
    StepExecution step = createAndSaveStep();
    int initialVersion = step.getVersion();

    repository.updateItemCount(step.getId(), 42);
    em.clear();

    StepExecution reloaded = em.find(StepExecution.class, step.getId());
    assertThat(reloaded.getVersion()).isEqualTo(initialVersion + 1);
    assertThat(reloaded.getItemCount()).isEqualTo(42);
}
```

**Sources**:
- [Thorben Janssen — Implementing Bulk Updates with Spring Data JPA](https://thorben-janssen.com/implementing-bulk-updates-with-spring-data-jpa/)
- [Baeldung — Spring Data JPA @Modifying Annotation](https://www.baeldung.com/spring-data-jpa-modifying-annotation)

---

## What NOT to Test

### Skip These (Framework-Validated)

| What | Why |
|------|-----|
| Inherited CRUD: `save()`, `findById()`, `delete()` | Tested by the Spring Data JPA team |
| Derived query compilation: `findByName()`, `findByStatusAndType()` | Spring Data validates at application startup |
| Schema generation | Use Flyway migrations instead; test migration execution as startup validation |

### DO Test These

- **Custom `@Query` methods** — both JPQL and native SQL; native queries are especially fragile
- **`@Modifying` queries** — bypass dirty-checking, must be verified
- **Projection interfaces/DTOs** — field mapping errors are common
- **Database constraints** — unique constraints, foreign keys, check constraints
- **Aggregate persistence round-trips** — save + clear + reload + verify all fields
- **Optimistic locking** — concurrent modification detection via `@Version`
- **Inheritance discriminator values** — correct subtype storage and retrieval

**Sources**:
- [Arho Huttunen — Testing the Persistence Layer](https://www.arhohuttunen.com/spring-boot-datajpatest/)
- [Reflectoring.io — Testing JPA Queries with @DataJpaTest](https://reflectoring.io/spring-boot-data-jpa-test/)

---

## Anti-Patterns Summary

| Anti-Pattern | Fix |
|---|---|
| `save()` then `findById()` without `flush()+clear()` | Always flush+clear before read assertions |
| Testing `save()` and `findById()` only | Test YOUR query methods, not Spring Data plumbing |
| Loading `@SpringBootTest` for isolated repository tests | Use `@DataJpaTest` — 90% less infrastructure |
| Using H2 for native queries with PostgreSQL syntax | Use Testcontainers with production DB image |
| Using the repository under test to set up test data | Use `TestEntityManager` for setup |
| Assuming `@Transactional` test behavior matches production | It doesn't — lazy loading, flush timing, and transaction boundaries differ |


---

# JPA Testing: Lazy Loading, Entity Lifecycle, Inheritance, and Read-Only Repositories

Covers lazy loading behavior in `org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest` (`@DataJpaTest`) tests vs production, N+1 detection, testing `SINGLE_TABLE` inheritance, and the `@QueryHint(HINT_READ_ONLY)` ClassCastException pitfall with read-only repositories.

---

## Lazy Loading in @DataJpaTest: The Hidden Trap

```java
import org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest;
import org.springframework.boot.jpa.test.autoconfigure.TestEntityManager;

@DataJpaTest
class OrderRepositoryTest {

    @Autowired TestEntityManager em;
    @Autowired OrderRepository orderRepository;

    @Test
    void findOrderWithItems_lazyLoadingInTransaction() {
        Customer customer = em.persist(new Customer("alice@example.com"));
        Order order = em.persist(new Order(customer));
        em.persist(new OrderItem(order, "PROD-1", 2));
        em.persist(new OrderItem(order, "PROD-2", 1));
        em.flush();
        em.clear();

        Order loaded = orderRepository.findById(order.getId()).orElseThrow();

        // This WORKS because @DataJpaTest wraps the test in @Transactional,
        // keeping the EntityManager open for the entire test.
        // In production without a transaction, this would throw LazyInitializationException.
        assertThat(loaded.getItems()).hasSize(2);
    }
}
```

**The problem**: `@OneToMany(fetch = LAZY)` collections load successfully in a `@DataJpaTest` because the test-level `@Transactional` keeps the `EntityManager` open. This masks `LazyInitializationException` that production code would encounter when accessed outside a transaction.

### Testing That LazyInitializationException IS Thrown in Production

To verify production-like behavior, use `@SpringBootTest` without `@Transactional`:

```java
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
class OrderLazyLoadingTest {

    @Autowired OrderRepository orderRepository;

    @Test
    void findOrder_accessingItemsOutsideTransaction_throwsLazyInitializationException() {
        // Setup: save order with items (done in a service or @Transactional helper)
        UUID savedId = createOrderWithItems();

        // Loading in @SpringBootTest does NOT wrap in @Transactional
        Order order = orderRepository.findById(savedId).orElseThrow();

        // Accessing lazy collection outside transaction should throw
        assertThatThrownBy(() -> order.getItems().size())
            .isInstanceOf(org.hibernate.LazyInitializationException.class);
    }
}
```

---

## N+1 Detection in Tests

N+1 problems are hard to see in `@DataJpaTest` because the test has one entity. Use a test with enough data to make the SQL count visible, combined with a query counter.

### Detecting N+1 with DataSource-Proxy

A SQL counter using DataSource-Proxy can assert the number of queries:

```java
import net.ttddyy.dsproxy.asserts.ProxyTestDataSource;

// In @BeforeEach, wrap the DataSource with DataSource-Proxy
// Then in the test:
@Test
void findOrders_noNPlusOne() {
    // Given: 10 orders with customers
    for (int i = 0; i < 10; i++) {
        Customer c = em.persist(new Customer("user" + i + "@example.com"));
        em.persist(new Order(c, OrderStatus.PENDING));
    }
    em.flush();
    em.clear();

    proxyDataSource.reset();
    List<Order> orders = orderRepository.findAllWithCustomer();

    // Should be 1 query (join fetch), not 11 (1 + N)
    assertThat(proxyDataSource.getQueryExecutions()).hasSize(1);
}
```

### Using JOIN FETCH to Solve N+1

```java
// Repository method using JOIN FETCH to prevent N+1:
@Query("SELECT o FROM Order o JOIN FETCH o.customer WHERE o.status = :status")
List<Order> findAllWithCustomer(@Param("status") OrderStatus status);
```

**Caveat**: Avoid `JOIN FETCH` with `SINGLE_TABLE` inheritance across multiple branches — see Hibernate 6.x `ClassCastException` in `references/hibernate-6-migration.md`.

---

## Testing Entity Relationships

### @OneToMany: Cascade and Orphan Removal

```java
@Test
void saveOrderWithItems_cascadesPersist() {
    Customer customer = em.persistAndFlush(new Customer("alice@example.com"));
    Order order = new Order(customer);
    order.addItem(new OrderItem("PROD-1", 2));
    order.addItem(new OrderItem("PROD-2", 1));

    orderRepository.save(order);
    em.flush();
    em.clear();

    Order loaded = orderRepository.findById(order.getId()).orElseThrow();
    // Access within transaction — safe in @DataJpaTest
    assertThat(loaded.getItems()).hasSize(2);
}

@Test
void deleteOrder_cascadesOrphanRemoval() {
    // Setup: order with items
    UUID orderId = createOrderWithItems(2);
    em.flush();
    em.clear();

    orderRepository.deleteById(orderId);
    em.flush();
    em.clear();

    // Verify items were also removed (CascadeType.REMOVE / orphanRemoval=true)
    assertThat(orderItemRepository.findByOrderId(orderId)).isEmpty();
}
```

### @ManyToOne: Foreign Key Integrity

```java
@Test
void persist_withNullMandatoryRelationship_throwsConstraintViolation() {
    Order order = new Order(null, OrderStatus.PENDING); // null customer is NOT NULL constraint

    assertThatThrownBy(() -> {
        em.persist(order);
        em.flush();
    }).isInstanceOf(org.springframework.dao.DataIntegrityViolationException.class);
}
```

---

## Testing SINGLE_TABLE Inheritance

### How SINGLE_TABLE Works

```java
import jakarta.persistence.DiscriminatorColumn;
import jakarta.persistence.DiscriminatorValue;
import jakarta.persistence.Entity;
import jakarta.persistence.Inheritance;
import jakarta.persistence.InheritanceType;

@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "type")
public abstract class JobExecution { ... }

@Entity
@DiscriminatorValue("COLLECTION")
public class CollectionExecution extends JobExecution { ... }
```

All entities in the hierarchy share one table. The `type` column differentiates rows.

### Testing SINGLE_TABLE Strategies

1. **Test polymorphic queries**: `findAll()` on a base repository returns both base and subclass instances.
2. **Test discriminator values**: Insert a subclass entity, query the raw table, verify the discriminator column.
3. **Test subclass-specific fields**: Subclass-specific columns are null for base class instances.
4. **Test subclass repository filtering**: A repository typed to a subclass only returns matching discriminator rows.
5. **Avoid join fetch across inheritance branches**: Hibernate 6 may throw `ClassCastException` — see `references/hibernate-6-migration.md`.

### Example Test Pattern

```java
import org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest;
import org.springframework.boot.jpa.test.autoconfigure.TestEntityManager;

@DataJpaTest
class JobExecutionRepositoryTest {

    @Autowired TestEntityManager em;
    @Autowired JobExecutionRepository jobExecutionRepository;

    @Test
    void findAll_shouldReturnSubtypeInstances() {
        CollectionExecution exec = new CollectionExecution();
        exec.setStatus(BatchStatus.STARTING);
        em.persistAndFlush(exec);
        em.clear();

        List<JobExecution> all = jobExecutionRepository.findAll();
        assertThat(all).hasSize(1);
        assertThat(all.get(0)).isInstanceOf(CollectionExecution.class);
    }

    @Test
    void collectionExecutionRepository_onlyReturnsCollectionType() {
        CollectionExecution collExec = new CollectionExecution();
        ReportExecution reportExec = new ReportExecution();
        em.persist(collExec);
        em.persist(reportExec);
        em.flush();
        em.clear();

        List<CollectionExecution> results = collectionExecutionRepository.findAll();
        assertThat(results).hasSize(1);
        assertThat(results.get(0)).isInstanceOf(CollectionExecution.class);
    }
}
```

**Sources**:
- [Baeldung — Query JPA Repository with Single Table Inheritance](https://www.baeldung.com/jpa-inheritance-single-table)
- [Vlad Mihalcea — The best way to map @DiscriminatorColumn](https://vladmihalcea.com/the-best-way-to-map-the-discriminatorcolumn-with-jpa-and-hibernate/)
- [Hibernate Discourse — ClassCastException with join fetch and inheritance](https://discourse.hibernate.org/t/classcastexception-in-hibernate-6-when-join-fetch-is-used-in-a-query-with-entity-inheritance/7815)

---

## Read-Only Repositories

### The Read-Only Repository Pattern

For DDD read models, extend `org.springframework.data.repository.Repository` (not `JpaRepository`) with only query methods. This enforces the aggregate boundary at compile time — callers cannot accidentally call `save()` or `delete()`.

```java
import org.springframework.data.repository.NoRepositoryBean;
import org.springframework.data.repository.Repository;

@NoRepositoryBean
public interface ReadOnlyRepository<T, ID> extends Repository<T, ID> {
    Optional<T> findById(ID id);
    List<T> findAll();
    long count();
}
```

### @QueryHint(HINT_READ_ONLY) ClassCastException — Hibernate 6.6.x

**Symptom**: `@QueryHint(name = HINT_READ_ONLY, value = "true")` on `findById` throws `ClassCastException` in Hibernate 6.6.x.

**Root cause**: Spring Data JPA implements `findById` using `EntityManager.find()`, which passes hints as `Map<String, Object>`. The `@QueryHints` annotation always passes values as `String`, but Hibernate 6.6 expects `Boolean` for the `org.hibernate.readOnly` hint with `em.find()`.

```java
// BROKEN in Hibernate 6.6.x:
@QueryHints(@QueryHint(name = org.hibernate.jpa.QueryHints.HINT_READ_ONLY, value = "true"))
Optional<T> findById(ID id); // ClassCastException: String cannot be cast to Boolean
```

### Preferred Fix: @Transactional(readOnly = true)

Since Spring Framework 5.1, `@Transactional(readOnly = true)` propagates the read-only flag to the Hibernate `Session` via `Session.setDefaultReadOnly(true)`. This:

- Disables dirty-checking for all loaded entities
- Skips hydrated state snapshots (memory savings)
- Works consistently with all query types including `em.find()`
- Avoids the String/Boolean type mismatch entirely

```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@Transactional(readOnly = true)
public class UserReadService {
    public List<User> findAll() {
        return repository.findAll();
    }
}
```

### Testing Read-Only Behavior

```java
@Test
void findById_readOnlyContext_doesNotDirtyCheck() {
    User saved = em.persistAndFlush(new User("Alice"));
    em.clear();

    // Load via read-only service
    User loaded = userReadService.findById(saved.getId()).orElseThrow();
    loaded.setEmail("modified@example.com"); // Modify in-memory

    em.flush(); // Should NOT generate an UPDATE statement (dirty-check disabled)

    em.clear();
    User reloaded = em.find(User.class, saved.getId());
    // Email should be unchanged because dirty-checking was disabled
    assertThat(reloaded.getEmail()).isEqualTo("Alice");
}
```

**Sources**:
- [Vlad Mihalcea — Spring read-only transaction Hibernate optimization](https://vladmihalcea.com/spring-read-only-transaction-hibernate-optimization/)
- [Thorben Janssen — Hibernate's Read-Only Query Hint](https://thorben-janssen.com/read-only-query-hint/)
- [Spring Data JPA #1503 — HINT_READONLY not applied to findOne()](https://github.com/spring-projects/spring-data-jpa/issues/1503)

---

## Optimistic Locking Tests

`@Version` fields protect against lost updates. Test that concurrent modification is detected:

```java
import jakarta.persistence.OptimisticLockException;

@Test
void optimisticLocking_concurrentModification_throwsOptimisticLockException() {
    Product saved = em.persistAndFlush(new Product("Widget", BigDecimal.TEN));
    UUID id = saved.getId();
    em.clear();

    // Load two separate instances (simulate two concurrent requests)
    Product instance1 = productRepository.findById(id).orElseThrow();
    Product instance2 = productRepository.findById(id).orElseThrow();

    // First save succeeds
    instance1.setPrice(new BigDecimal("15.00"));
    productRepository.saveAndFlush(instance1);
    em.clear();

    // Second save should fail — stale version
    instance2.setPrice(new BigDecimal("20.00"));
    assertThatThrownBy(() -> productRepository.saveAndFlush(instance2))
        .isInstanceOf(OptimisticLockException.class)
        .extracting(e -> ((OptimisticLockException) e).getEntity())
        .isEqualTo(instance2);
}
```

**Note**: This test requires a `@Transactional(propagation = NOT_SUPPORTED)` context or `@SpringBootTest` — optimistic lock conflicts require separate transaction commits, which the `@DataJpaTest` automatic rollback prevents.


---

# JPA Testing: Hibernate 6.x / 7.x Migration Pitfalls and Spring Boot 4 Package Changes

Covers the Hibernate 6.x and 7.x migration pitfalls that affect test code specifically, plus the Spring Boot 3.x → 4.x test autoconfigure package reorganization. All package names here are Spring Boot 4 / Spring Framework 7.

---

## Spring Boot 3.x → 4.x: Test Package Reorganization

All test autoconfigure packages moved in Spring Boot 4. Using Boot 3 import paths in Boot 4 projects will cause `ClassNotFoundException` at compile time.

| Annotation / Class | Boot 3.x Package | Boot 4.x Package |
|---|---|---|
| `@DataJpaTest` | `org.springframework.boot.test.autoconfigure.orm.jpa` | `org.springframework.boot.data.jpa.test.autoconfigure` |
| `@AutoConfigureTestDatabase` | `org.springframework.boot.test.autoconfigure.orm.jpa` | `org.springframework.boot.data.jpa.test.autoconfigure` |
| `TestEntityManager` | `org.springframework.boot.test.autoconfigure.orm.jpa` | `org.springframework.boot.jpa.test.autoconfigure` |
| `@WebMvcTest` | `org.springframework.boot.test.autoconfigure.web.servlet` | `org.springframework.boot.webmvc.test.autoconfigure` |
| `@WebFluxTest` | `org.springframework.boot.test.autoconfigure.web.reactive` | `org.springframework.boot.webflux.test.autoconfigure` |
| `@MockBean` | `org.springframework.boot.test.mock.mockito` | Replaced by `@MockitoBean` |
| `@MockitoBean` | *(did not exist in Boot 3)* | `org.springframework.test.context.bean.override.mockito` |

**Note on `@MockBean` → `@MockitoBean`**: Spring Framework 7 introduced `org.springframework.test.context.bean.override.mockito.MockitoBean` (`@MockitoBean`). The Boot 3 `@MockBean` from `org.springframework.boot.test.mock.mockito` is removed in Boot 4. Do a global find-and-replace of `import org.springframework.boot.test.mock.mockito.MockBean` → `import org.springframework.test.context.bean.override.mockito.MockitoBean` and `@MockBean` → `@MockitoBean`.

**`@SpringBootTest` and `@ServiceConnection` are unchanged**: These remain at `org.springframework.boot.test.context.SpringBootTest` and `org.springframework.boot.testcontainers.service.connection.ServiceConnection`.

### Quick Migration Checklist for Test Code

```bash
# Find all Boot 3 test autoconfigure imports to update
grep -r "org.springframework.boot.test.autoconfigure" src/test/
grep -r "org.springframework.boot.test.mock.mockito.MockBean" src/test/
```

---

## Hibernate 6.x Migration Pitfalls

### 5.1 Sequence Generator Naming (Hibernate 6.0)

Hibernate 6 changed the default sequence naming strategy. Previously, all entities shared `hibernate_sequence`. Now each entity gets `<table_name>_SEQ` by default.

**Impact on tests**: Test data scripts that use `nextval('hibernate_sequence')` break. Either hardcode IDs or update sequence references.

```sql
-- Before (Hibernate 5.x):
INSERT INTO orders (id, ...) VALUES (nextval('hibernate_sequence'), ...);

-- After (Hibernate 6.x):
INSERT INTO orders (id, ...) VALUES (nextval('orders_SEQ'), ...);
```

To restore the old behavior (not recommended — breaks multi-entity sequences):
```properties
spring.jpa.properties.hibernate.id.new_generator_mappings=false
```

**Source**: [Thorben Janssen — 8 Things to Know When Migrating to Hibernate 6.x](https://thorben-janssen.com/things-to-know-when-migrating-to-hibernate-6-x/)

---

### 5.2 Jakarta Persistence Package Migration (Hibernate 6.0)

All `javax.persistence.*` imports changed to `jakarta.persistence.*`. This is a global find-and-replace but affects all test code, `TestEntityManager` usage, and SQL scripts that reference JPA-mapped types.

```java
// Before (javax):
import javax.persistence.EntityManager;
import javax.persistence.PersistenceContext;

// After (jakarta):
import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
```

Also affects `@Entity`, `@Id`, `@Column`, `@OneToMany`, `@Transactional` (the JPA one), etc.

**Note**: `org.springframework.transaction.annotation.Transactional` (Spring's `@Transactional`) did NOT change — only the JPA `javax.transaction.Transactional` moved to `jakarta.transaction.Transactional`.

---

### 5.3 HQL Column Names Disallowed (Hibernate 6.0)

Hibernate 6 only accepts JPA attribute names in HQL/JPQL — database column names are rejected. Tests with custom JPQL that used column names instead of field names will fail.

```java
// BROKEN in Hibernate 6 (uses column name "first_name" instead of field name "firstName"):
@Query("SELECT u FROM User u WHERE u.first_name = :name")

// FIXED (use JPA attribute name):
@Query("SELECT u FROM User u WHERE u.firstName = :name")
```

Symptom: `QueryException: No data was found for query ... first_name is not mapped`.

---

### 5.4 Instant and Duration Mapping Changes (Hibernate 6.0+)

`Instant` and `Duration` type mappings changed between Hibernate 5 and 6. Entities using these types may behave differently if the schema was generated by an older Hibernate version.

Common symptom: timezone handling differences for `Instant` fields when migrating from `TIMESTAMP` without timezone to `TIMESTAMP WITH TIME ZONE`.

---

### 5.5 ClassCastException with JOIN FETCH and Inheritance (Hibernate 6.0+)

A known Hibernate 6 issue: queries using `JOIN FETCH` with entity inheritance hierarchies can throw `ClassCastException` when multiple inheritance branches are fetched in a single query.

**Symptom**:
```
java.lang.ClassCastException: class CollectionExecution cannot be cast to class ReportExecution
```

**Workaround**: Avoid `JOIN FETCH` across multiple inheritance branches. Use subclass-specific repositories instead, or load the inheritance hierarchy in separate queries.

For `SINGLE_TABLE` inheritance testing, fetch one discriminator type at a time:

```java
// PROBLEMATIC: JOIN FETCH with mixed inheritance branches
@Query("SELECT j FROM JobExecution j JOIN FETCH j.steps")
List<JobExecution> findAllWithSteps();

// SAFER: fetch by concrete subtype
@Query("SELECT c FROM CollectionExecution c JOIN FETCH c.steps")
List<CollectionExecution> findCollectionsWithSteps();
```

**Source**: [Hibernate Discourse — ClassCastException in Hibernate 6 when join fetch is used with entity inheritance](https://discourse.hibernate.org/t/classcastexception-in-hibernate-6-when-join-fetch-is-used-in-a-query-with-entity-inheritance/7815)

---

### 5.6 @CreationTimestamp + @UuidGenerator Pitfall (Hibernate 6.x)

**Discovered in tuvium-collector Step 3.8**.

`org.hibernate.annotations.UuidGenerator` sets the entity ID at `persist()` time. Subsequent `save()` calls on an already-persisted entity trigger `merge()` (not `persist()`), which can produce a spurious `UPDATE` with a `null` `createTime` if `@CreationTimestamp` is used.

**Root cause**: `merge()` does not trigger `@CreationTimestamp` initialization. If `createTime` is null at merge time, Hibernate writes null to the database.

**Fix**: Initialize timestamp fields with a default value in the field declaration:

```java
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UuidGenerator;

@Entity
public class AuditedEntity {

    @Id
    @UuidGenerator
    private UUID id;

    @CreationTimestamp
    private Instant createTime = Instant.now(); // REQUIRED: initialize for merge() safety

    @UpdateTimestamp
    private Instant updateTime = Instant.now(); // Same fix applies
}
```

**Test pattern that exposes this bug**:

```java
@Test
void save_twiceOnSameEntity_preservesCreationTimestamp() {
    AuditedEntity entity = new AuditedEntity();
    AuditedEntity first = repository.saveAndFlush(entity);
    em.clear();

    // Reload and save again (simulates service calling save() on a detached entity)
    AuditedEntity reloaded = repository.findById(first.getId()).orElseThrow();
    reloaded.setName("Updated");
    AuditedEntity second = repository.saveAndFlush(reloaded);
    em.clear();

    AuditedEntity final_ = repository.findById(first.getId()).orElseThrow();
    assertThat(final_.getCreateTime()).isNotNull(); // Fails without Instant.now() default
    assertThat(final_.getCreateTime()).isEqualTo(first.getCreateTime()); // Same creation time
}
```

---

### 5.7 @QueryHint HINT_READ_ONLY String/Boolean Issue (Hibernate 6.6.x)

**Discovered in tuvium-collector** (see also `references/lazy-loading-tests.md` for the fix).

`@QueryHint(name = HINT_READ_ONLY, value = "true")` on `findById` causes `ClassCastException` in Hibernate 6.6.x.

**Root cause**: Spring Data JPA's `findById` uses `EntityManager.find()`, which passes hints as `Map<String, Object>`. The `@QueryHints` annotation always passes values as `String`, but Hibernate 6.6 requires `Boolean` for `org.hibernate.readOnly` when used with `em.find()`.

```java
// BROKEN in Hibernate 6.6.x:
@QueryHints(@QueryHint(name = org.hibernate.jpa.QueryHints.HINT_READ_ONLY, value = "true"))
Optional<T> findById(ID id);
// ClassCastException: java.lang.String cannot be cast to java.lang.Boolean
```

**Fix**: Use `@Transactional(readOnly = true)` on the service layer instead:

```java
@Service
@Transactional(readOnly = true)
public class ReadService {
    public Optional<T> findById(ID id) {
        return repository.findById(id);
    }
}
```

Since Spring Framework 5.1, `@Transactional(readOnly = true)` propagates to the Hibernate `Session` via `Session.setDefaultReadOnly(true)`, providing the same optimization without the type mismatch.

**Related issues**:
- [Spring Data JPA #1503 — HINT_READONLY not applied to findOne()](https://github.com/spring-projects/spring-data-jpa/issues/1503)
- [Spring Framework #21494 — Propagate read-only to Hibernate Session](https://github.com/spring-projects/spring-framework/issues/21494)

---

## Hibernate 7.x Notes (Spring Boot 4)

Spring Boot 4 ships with Hibernate 7.x. Hibernate 7 is largely a continuation of Hibernate 6.x with refinements:

- Jakarta persistence requirement continues (no `javax.persistence` support)
- Sequence naming strategy from 6.0 remains (`<table_name>_SEQ`)
- HQL column name restriction from 6.0 remains
- The `@CreationTimestamp` + `@UuidGenerator` initialization fix (5.6) applies equally

Check the [Hibernate 7.0 Migration Guide](https://docs.hibernate.org/orm/7.0/migration-guide/) for new pitfalls specific to 7.0.

---

## Migration Checklist for Tests

When migrating from Boot 3 + Hibernate 5/6 to Boot 4 + Hibernate 7:

- [ ] Update all `org.springframework.boot.test.autoconfigure.orm.jpa.*` imports to Boot 4 packages
- [ ] Replace `@MockBean` with `@MockitoBean` from `org.springframework.test.context.bean.override.mockito`
- [ ] Grep for `javax.persistence` and replace with `jakarta.persistence`
- [ ] Grep for `nextval('hibernate_sequence')` in test SQL scripts
- [ ] Grep for HQL/JPQL using column names (e.g., `snake_case` in queries against `camelCase` fields)
- [ ] Check entities with `@UuidGenerator` + `@CreationTimestamp` — add `= Instant.now()` defaults
- [ ] Remove `@QueryHint(HINT_READ_ONLY)` on `findById` — use `@Transactional(readOnly = true)` instead
- [ ] Test `SINGLE_TABLE` inheritance queries — avoid `JOIN FETCH` across multiple branches

**Sources**:
- [Thorben Janssen — 8 Things to Know When Migrating to Hibernate 6.x](https://thorben-janssen.com/things-to-know-when-migrating-to-hibernate-6-x/)
- [Hibernate 6.0 Migration Guide](https://docs.hibernate.org/orm/6.0/migration-guide/)
- [Hibernate 6.6 Migration Guide](https://docs.jboss.org/hibernate/orm/6.6/migration-guide/migration-guide.html)
